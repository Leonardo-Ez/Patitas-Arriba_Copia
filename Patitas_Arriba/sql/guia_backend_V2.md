# Guía de la base de datos V2 para el backend

Documento de referencia sobre `Patitas_ArribaDDL_V2.sql`: qué cambió, cómo está modelado y qué reglas **no garantiza la base** y debe implementar el backend.

> Este documento reemplaza en lo que contradiga a `analisis_activo_V2.md`, que se escribió antes de varias decisiones (por ejemplo, ya no existe `CATEGORIA_ARTICULO` y `ATENCION_MEDICA` usa `ANULADO`).

---

## 1. Qué cambió respecto a `patitasarriba-mysql.sql` (versión anterior)

### Tablas
| Cambio | Detalle |
|---|---|
| Eliminadas | `CATEGORIA_ARTICULO`, `TRATAMIENTO`, `ATENCION_TRATAMIENTO` |
| Nuevas | `CATEGORIA` y `ARTICULO_CATEGORIA` (relación N:M entre artículo y categoría) |

### Columnas y tipos
| Tabla | Cambio |
|---|---|
| `CUENTA` | `PASSWORD` pasa a `VARCHAR(255)`. `NOMBRE_USUARIO` y `CORREO` son UNIQUE. |
| `CLIENTE` | `ID_CUENTA` admite NULL (cliente sin cuenta). |
| `BOLETA` | `ACTIVO` se reemplaza por `ANULADO` (`DEFAULT 0`). |
| `ARTICULO` | Se quita `ID_CATEGORIA_ARTICULO` y `MARCA`. `DESCRIPCION` pasa a `VARCHAR(200) NULL`. `STOCK_ACTUAL` es `INT UNSIGNED`. |
| `MASCOTA` | `SEXO` pasa de `CHAR(1)` a `ENUM('M','H')`. |
| `CITA` | Se quita `ACTIVO`. `ESTADO` pasa a `PENDIENTE, APROBADO, RECHAZADO, CANCELADO, COMPLETADO`. Se agrega `ES_EMERGENCIA TINYINT(1) DEFAULT 0`. |
| `SERVICIO` | El enum `TIPO_SERVICIO_MEDICO` queda en `CONSULTA_MEDICA, OPERACION, VACUNACION` (sin `ESTETICO` ni `EMERGENCIA`). `DESCRIPCION` pasa a 200. |
| `ATENCION_MEDICA` | `ID_CITA_MEDICA` se reemplaza por `ID_DETALLE_CITA` (con índice, sin UNIQUE). `ACTIVO` se reemplaza por `ANULADO` (`DEFAULT 0`). Se amplían `MOTIVO_CONSULTA` (200) y `OBSERVACIONES` (500). |
| `DETALLE_BOLETA_ART` | Nuevo `PRECIO_UNITARIO`. `SUBTOTAL` es **columna generada**. Se quita `ACTIVO`. |
| `DETALLE_BOLETA_SERV` | `ID_SERVICIO` se reemplaza por `ID_DETALLE_CITA` (con índice, sin UNIQUE). Nuevo `PRECIO_UNITARIO`. `SUBTOTAL` es columna generada. Se quita `ACTIVO`. |
| `DETALLE_CITA`, `DETALLE_RECETA`, `ATENCION_DIAGNOSTICO`, `HORARIO_ADMINISTRADOR`, `HORARIO_VETERINARIO`, `INSUMO_UTILIZADO` | Se quita `ACTIVO`. |
| `INSUMO_UTILIZADO` | `ID_ATENCION_TRATAMIENTO` se reemplaza por `ID_ATENCION_MEDICA`. |
| `DIAGNOSTICO` | `DESCRIPCION` pasa a 200. |

### Restricciones nuevas
- `VETERINARIO.NUMERO_COLEGIATURA` UNIQUE.
- `DETALLE_CITA (ID_CITA, ID_SERVICIO)` UNIQUE: un servicio no se repite dentro de la misma cita.
- `ATENCION_DIAGNOSTICO (ID_ATENCION_MEDICA, ID_DIAGNOSTICO)` UNIQUE.
- `HORARIO_VETERINARIO (ID_VETERINARIO, ID_HORARIO)` y `HORARIO_ADMINISTRADOR (ID_ADMINISTRADOR, ID_HORARIO)` UNIQUE.
- `DETALLE_BOLETA_ART (ID_BOLETA, ID_ARTICULO)` UNIQUE.
- `ATENCION_MEDICA.ID_DETALLE_CITA` y `DETALLE_BOLETA_SERV.ID_DETALLE_CITA` **no son UNIQUE** (solo tienen índice). Ver 4.3 y 4.5.
- `CATEGORIA.NOMBRE` UNIQUE.

---

## 2. Mapa del modelo

```
CUENTA ──1:0..1── CLIENTE ──1:N── MASCOTA ──1:N── CITA ──1:N── DETALLE_CITA ──N:1── SERVICIO
   │                  │                              │              │
   ├──1:1── VETERINARIO ──1:N── CITA                 │              ├──1:N── ATENCION_MEDICA* ──1:N── RECETA ──1:N── DETALLE_RECETA ──N:1── ARTICULO
   └──1:1── ADMINISTRADOR                            │              │              ├──1:N── ATENCION_DIAGNOSTICO ──N:1── DIAGNOSTICO
                                                     │              │              └──1:N── INSUMO_UTILIZADO ──N:1── ARTICULO
CLIENTE ──1:N── BOLETA ──1:N── DETALLE_BOLETA_ART ──N:1── ARTICULO ──N:M── CATEGORIA (vía ARTICULO_CATEGORIA)
                       └─1:N── DETALLE_BOLETA_SERV* ──N:1── DETALLE_CITA
HORARIO ──N:M── VETERINARIO (HORARIO_VETERINARIO) y ──N:M── ADMINISTRADOR (HORARIO_ADMINISTRADOR)

* Por diseño debe haber una sola atención vigente y una sola línea de cobro vigente por `DETALLE_CITA`,
  pero lo controla el backend (ver 4.3 y 4.5), no la base.
```

Dos flujos centrales:

1. **Clínico:** `CITA` → `DETALLE_CITA` (uno por servicio) → `ATENCION_MEDICA` (una vigente por detalle) → `RECETA` / `ATENCION_DIAGNOSTICO` / `INSUMO_UTILIZADO`.
2. **Cobro:** los **servicios solo se cobran a través de una cita** (`DETALLE_BOLETA_SERV.ID_DETALLE_CITA`). Los **artículos se venden directamente** (`DETALLE_BOLETA_ART`), sin necesidad de cita.

---

## 3. Convenciones de estado: `ACTIVO`, `ANULADO` y `ESTADO`

| Mecanismo | Significado | Dónde |
|---|---|---|
| `ACTIVO = 1` | Registro vigente. `0` es una baja lógica. | `CUENTA`, `CLIENTE`, `MASCOTA`, `ADMINISTRADOR`, `VETERINARIO`, `HORARIO`, `ARTICULO`, `SERVICIO`, `CATEGORIA`, `DIAGNOSTICO`, `RECETA` |
| `ANULADO = 0` | Documento vigente. `1` es un documento anulado. **Es lo contrario de `ACTIVO`.** | `BOLETA`, `ATENCION_MEDICA` |
| `ESTADO` | Flujo con varios pasos. | `CITA` |
| Nada | Tablas de detalle, intermedias o de consumo. Su vida depende del padre. | resto |

**En el backend:**
- Los filtros "solo vigentes" son `ACTIVO = 1` en unas tablas y `ANULADO = 0` en otras. Conviene centralizarlo en un solo lugar para no confundirlos.
- Las FK son `NO ACTION`: **no hay borrado en cascada**. Un `DELETE` físico de un padre con hijos falla. Se usa baja lógica.
- Los detalles se borran físicamente solo mientras el padre es editable (por ejemplo, una cita `PENDIENTE`).

---

## 4. Reglas que el backend debe implementar

### 4.1 Cuentas y roles
- `CUENTA` no tiene rol. El rol se deduce de qué tabla la referencia (`CLIENTE`, `VETERINARIO` o `ADMINISTRADOR`).
- **La base no impide que una cuenta esté en dos roles a la vez.** El backend debe verificar, al crear cliente, veterinario o administrador, que la cuenta no esté ya asociada a otro rol.
- `CLIENTE.ID_CUENTA` puede ser NULL. Un cliente sin cuenta (por ejemplo, registrado en recepción) es un caso normal. No asumir que siempre hay cuenta.
- `CUENTA.ACTIVO` controla el acceso al sistema. El `ACTIVO` de la persona controla si aparece en listados y agendas. Son independientes. Si se desactiva una persona, hay que desactivar también su cuenta (o definir qué manda).
- Guardar solo el **hash** de la contraseña. La columna admite 255 caracteres, suficiente para bcrypt (60) y para Argon2.
- Normalizar `CORREO` y `NOMBRE_USUARIO` (por ejemplo, a minúsculas) antes de insertar. Con la collation por defecto el UNIQUE no distingue mayúsculas (`A@x.com` y `a@x.com` cuentan como duplicado), pero normalizar evita confusiones al mostrar y comparar.
- `DNI` (8) y `TELEFONO` (9) **no tienen CHECK**. Se validan formato y longitud en el backend.

### 4.2 Citas
- Estados válidos: `PENDIENTE`, `APROBADO`, `RECHAZADO`, `CANCELADO`, `COMPLETADO`. Flujo sugerido:
  `PENDIENTE → APROBADO | RECHAZADO`, `APROBADO → COMPLETADO | CANCELADO`. La base no valida las transiciones.
- `ES_EMERGENCIA` marca la emergencia. Ya no existe el servicio de tipo `EMERGENCIA` en el enum.
- **No hay protección contra doble reserva.** Dos citas del mismo veterinario con la misma `FECHA_HORA` son válidas para la base. Validar disponibilidad en el backend, ignorando citas `RECHAZADO` y `CANCELADO`.
- **No hay relación entre `CITA` y `HORARIO_VETERINARIO`.** Validar que la fecha caiga dentro del horario del veterinario.
- Una cita tiene uno o más `DETALLE_CITA` (un servicio cada uno). El mismo servicio no se puede repetir en una cita.
- Mascota y veterinario deben estar activos al agendar.

### 4.3 Atención médica
- Una atención corresponde a **un `DETALLE_CITA`**. Una cita con tres servicios genera hasta tres atenciones.
- **La base no impide registrar dos atenciones vigentes para el mismo `DETALLE_CITA`.** El backend debe comprobar, antes de insertar, que no exista otra con `ANULADO = 0`. Si la hay, rechazar. Si solo hay anuladas, se puede insertar una nueva.
- `ATENCION_MEDICA.ID_MASCOTA` se guarda **a propósito** para consultar el historial de una mascota con pocos JOIN. **Siempre debe coincidir** con la mascota de la cita (`DETALLE_CITA → CITA → ID_MASCOTA`). La base no lo verifica: el backend debe tomar el valor desde la cita y no aceptarlo desde el cliente.
- `ANULADO` vale 0 por defecto. Al anular una atención, **la base no propaga nada** a sus recetas, diagnósticos ni insumos. Hay que decidir y aplicar una regla:
  - no permitir anular una atención con recetas o diagnósticos, o
  - desactivar los hijos en la misma transacción, o
  - filtrar siempre por `am.ANULADO = 0` al leer los hijos.
- Sugerencia de flujo: crear la atención solo con la cita `APROBADO`. Cuando todos los `DETALLE_CITA` tengan atención, pasar la cita a `COMPLETADO`.
- `FECHA_HORA` de la atención es independiente de la de la cita (la real puede diferir de la agendada).

### 4.4 Receta, diagnóstico e insumos
- `DIAGNOSTICO` es un catálogo. El vínculo con la atención está en `ATENCION_DIAGNOSTICO`, y el mismo diagnóstico no se repite en una atención.
- `DETALLE_RECETA` y `INSUMO_UTILIZADO` referencian `ARTICULO` sin distinguir tipo. **La base permite recetar o consumir cualquier artículo.** Si solo ciertos artículos son medicamentos o insumos, validar en el backend (por ejemplo, por categoría).
- **Decisión pendiente:** ¿recetar descuenta stock? Hoy la base no lo hace ni lo exige. El stock baja cuando se vende (boleta) y cuando se registra un insumo utilizado.

### 4.5 Facturación (lo más delicado)

**Servicios:**
- Se cobran por `DETALLE_CITA`, no por servicio. Para cobrar una línea hay que indicar el `ID_DETALLE_CITA`.
- `PRECIO_UNITARIO` es una **copia** de `SERVICIO.PRECIO_BASE` en el momento del cobro. El backend debe leerlo del servicio y guardarlo. Así, si el precio base cambia, las boletas antiguas no se alteran.
- Un `DETALLE_CITA` debe facturarse **una sola vez en boletas vigentes**. **La base no lo impide**: el backend debe comprobar, antes de insertar la línea, que ese `ID_DETALLE_CITA` no esté en una boleta con `ANULADO = 0`. Hacer la comprobación y el `INSERT` en la misma transacción, bloqueando la fila con `SELECT ... FOR UPDATE` sobre el `DETALLE_CITA`, para que dos cobros simultáneos no pasen ambos.
- `CANTIDAD` en líneas de servicio vale siempre 1. Hoy es casi redundante.

**Artículos:**
- Se venden sin cita. `PRECIO_UNITARIO` se copia de `ARTICULO.PRECIO_BASE` igual que en los servicios.
- No se repite el mismo artículo en una boleta (UNIQUE). Si el cliente pide más, se aumenta `CANTIDAD`.

**`SUBTOTAL` es columna generada** (`CANTIDAD * PRECIO_UNITARIO`):
- **No se incluye en el `INSERT` ni en el `UPDATE`.** MySQL devuelve error 3105 si se intenta escribir un valor.
- Se lee normalmente con `SELECT`.
- Si cambia `CANTIDAD` o `PRECIO_UNITARIO`, se recalcula solo.

**`BOLETA.TOTAL` no se calcula en la base.** Es la suma de los `SUBTOTAL` de ambos detalles. El backend lo calcula y lo guarda **en la misma transacción** que crea la boleta y sus líneas. Si se modifican líneas después, hay que recalcularlo.

**Otras validaciones de facturación:**
- `BOLETA.ID_CLIENTE` debería coincidir con el dueño de la mascota de la cita (`CITA → MASCOTA → CLIENTE`). La base no lo comprueba.
- Cobrar solo servicios de citas en estado coherente (por ejemplo, `COMPLETADO`) y con atención no anulada. La base no lo exige.
- Usar tipos decimales exactos (por ejemplo, `BigDecimal` en Java) para dinero. Nunca `float` ni `double`.

> **Anulación de boletas y atenciones.** Se quitaron los UNIQUE sobre `ID_DETALLE_CITA` para poder volver a cobrar una cita o registrar otra atención después de anular. A cambio, **la unicidad entre registros vigentes la garantiza solo el backend** (ver 4.3 y arriba). Las líneas de una boleta anulada se conservan como historial.

### 4.6 Inventario
- `ARTICULO.STOCK_ACTUAL` es `INT UNSIGNED`: un descuento que lo deje bajo cero **falla** con error (en MySQL, el 1690 "value is out of range") en lugar de quedar negativo.
- Descontar con una sentencia atómica y dentro de la transacción de la venta, por ejemplo:
  `UPDATE ARTICULO SET STOCK_ACTUAL = STOCK_ACTUAL - ? WHERE ID_ARTICULO = ?`
  y manejar el error como "stock insuficiente". No leer el stock, restar en memoria y escribirlo.
- Al **anular una boleta**, el stock **no se repone solo**. El backend debe devolverlo.
- Al registrar `INSUMO_UTILIZADO`, descontar el stock en la misma transacción. Al anular una atención, decidir si se repone el stock de sus insumos: la base tampoco lo hace.
- `STOCK_MINIMO` es informativo. La alerta de reposición (`STOCK_ACTUAL <= STOCK_MINIMO`) se calcula en el backend.
- `STOCK_MINIMO` sigue con signo: validar que no sea negativo.

### 4.7 Horarios
- `HORARIO` define franjas (`DIA_SEMANA`, `HORA_INICIO`, `HORA_FIN`) y se asigna a veterinarios y administradores. No se puede asignar el mismo horario dos veces a la misma persona (UNIQUE).
- **`DIA_SEMANA` es texto libre** (`VARCHAR(10)`). Acordar un formato único (por ejemplo, `LUNES` en mayúsculas) y usar un enum en el backend.
- Validar `HORA_FIN > HORA_INICIO`.
- La base no impide que franjas distintas de una misma persona se solapen.

### 4.8 Catálogos
- `ARTICULO` ↔ `CATEGORIA` es N:M. Un artículo puede quedar sin categorías.
- `ARTICULO_CATEGORIA` no tiene `ACTIVO`: se agrega y se quita con `INSERT` y `DELETE`.
- Ya no existe `MARCA` en `ARTICULO`. Los modelos y formularios deben dejar de usarla.

---

## 5. Errores de base de datos que el backend debe traducir

| Situación | Código MySQL | Respuesta sugerida |
|---|---|---|
| Duplicado en UNIQUE (DNI, correo, usuario, colegiatura, servicio repetido en una cita, etc.) | 1062 | Conflicto: ya existe. Identificar la restricción por su nombre. |
| Inserción con FK inexistente | 1452 | Referencia no válida. |
| Borrado de un padre con hijos | 1451 | No se puede eliminar: tiene registros asociados. Usar baja lógica. |
| Escritura en columna generada (`SUBTOTAL`) | 3105 | Error de programación: quitar la columna del `INSERT`/`UPDATE`. |
| Stock que queda negativo | 1690 | Stock insuficiente. |
| Valor fuera del enum (`SEXO`, `ESTADO`, `METODO_PAGO`, etc.) | 1265 | Dato inválido. Evitarlo mapeando enums del backend a los de la base. |

Los errores 1690 y 1265 solo se producen con el modo SQL estricto (`STRICT_TRANS_TABLES`), que es el valor por defecto en MySQL 8. El script solo lo fija durante su propia ejecución, así que el servidor donde se despliegue debe tenerlo activo. Sin él, MySQL recorta o ajusta el valor sin avisar.

---

## 6. Impacto en el código existente

Revisar los DAO, modelos y consultas que usen cualquiera de esto:

| Dejó de existir o cambió | Acción |
|---|---|
| `ARTICULO.MARCA`, `ARTICULO.ID_CATEGORIA_ARTICULO` | Quitar. Las categorías se consultan por `ARTICULO_CATEGORIA`. |
| Tablas `TRATAMIENTO`, `ATENCION_TRATAMIENTO`, `CATEGORIA_ARTICULO` | Eliminar clases, DAO y consultas. |
| `ATENCION_MEDICA.ID_CITA_MEDICA` | Reemplazar por `ID_DETALLE_CITA`. |
| `INSUMO_UTILIZADO.ID_ATENCION_TRATAMIENTO` | Reemplazar por `ID_ATENCION_MEDICA`. |
| `DETALLE_BOLETA_SERV.ID_SERVICIO` | Reemplazar por `ID_DETALLE_CITA`. El servicio se obtiene por `DETALLE_CITA`. |
| `ACTIVO` en `CITA`, detalles, tablas intermedias e `INSUMO_UTILIZADO` | Quitar de modelos y consultas. |
| `BOLETA.ACTIVO`, `ATENCION_MEDICA.ACTIVO` | Ahora son `ANULADO`, con sentido contrario. |
| `DETALLE_BOLETA_*.SUBTOTAL` | Dejar de enviarlo en los `INSERT`. |
| Estados de `CITA` (`AGENDADA`, `COMPLETADA`, `CANCELADA`) | Ahora son `PENDIENTE`, `APROBADO`, `RECHAZADO`, `CANCELADO`, `COMPLETADO`. |
| `SERVICIO.TIPO_SERVICIO_MEDICO` con `ESTETICO` o `EMERGENCIA` | Valores eliminados. |
| `MASCOTA.SEXO` | Solo `M` o `H`. |

**Sin migración.** Es un reajuste del diseño y todavía no hay nada desplegado, así que no se necesita script de migración. Para aplicar el V2 se parte de una base vacía: se elimina el esquema anterior y se ejecuta `Patitas_ArribaDDL_V2.sql`. El script usa `CREATE TABLE IF NOT EXISTS`, así que **no actualiza** tablas que ya existan con la estructura vieja. Si queda alguna base de pruebas con la versión anterior, hay que borrarla antes (`DROP SCHEMA mydb`).

---

## 7. Consultas útiles

**Diagnósticos de una mascota** (usa `ATENCION_MEDICA.ID_MASCOTA` directamente):
```sql
SELECT am.FECHA_HORA, d.NOMBRE_ENFERMEDAD, ad.NIVEL_GRAVEDAD, ad.DETALLE_DIAGNOSTICO
FROM ATENCION_MEDICA am
JOIN ATENCION_DIAGNOSTICO ad ON ad.ID_ATENCION_MEDICA = am.ID_ATENCION_MEDICA
JOIN DIAGNOSTICO d           ON d.ID_DIAGNOSTICO = ad.ID_DIAGNOSTICO
WHERE am.ID_MASCOTA = ? AND am.ANULADO = 0
ORDER BY am.FECHA_HORA DESC;
```

**Servicios de una cita pendientes de cobro y su precio** (ignora las líneas de boletas anuladas):
```sql
SELECT dc.ID_DETALLE_CITA, s.NOMBRE, s.PRECIO_BASE
FROM DETALLE_CITA dc
JOIN SERVICIO s ON s.ID_SERVICIO = dc.ID_SERVICIO
WHERE dc.ID_CITA = ?
  AND NOT EXISTS (
        SELECT 1
        FROM DETALLE_BOLETA_SERV dbs
        JOIN BOLETA b ON b.ID_BOLETA = dbs.ID_BOLETA
        WHERE dbs.ID_DETALLE_CITA = dc.ID_DETALLE_CITA
          AND b.ANULADO = 0);
```
La misma condición `NOT EXISTS` sirve para la comprobación previa al cobro descrita en 4.5.

**Recetas vigentes de una mascota:**
```sql
SELECT r.ID_RECETA, r.FECHA_EMISION, r.INDICACIONES_GENERALES
FROM RECETA r
JOIN ATENCION_MEDICA am ON am.ID_ATENCION_MEDICA = r.ID_ATENCION_MEDICA
WHERE am.ID_MASCOTA = ? AND r.ACTIVO = 1 AND am.ANULADO = 0;
```

**Artículos bajo el stock mínimo:**
```sql
SELECT ID_ARTICULO, NOMBRE, STOCK_ACTUAL, STOCK_MINIMO
FROM ARTICULO
WHERE ACTIVO = 1 AND STOCK_ACTUAL <= STOCK_MINIMO;
```

---

## 8. Decisiones de diseño intencionales

Para que nadie las "corrija" por error:

- `ATENCION_MEDICA.ID_MASCOTA` se mantiene aunque se pueda deducir de la cita: sirve para consultar el historial sin tantos JOIN.
- Cada `DETALLE_CITA` tiene una sola atención **vigente** y una sola línea de cobro **vigente**. Se controla en el backend, no con UNIQUE, para poder rehacer tras una anulación.
- `CLIENTE.ID_CUENTA` admite NULL.
- El borrado lógico se aplica solo a entidades maestras y documentos (ver sección 3).
- Los servicios solo se cobran a través de una cita. Los artículos sí se pueden vender sin cita.
- Las emergencias se marcan solo con `CITA.ES_EMERGENCIA`.
- `SERVICIO.REQUIERE_TRIAJE` y `REQUIERE_VACUNA` quedan como banderas. No hay tablas para registrar triaje ni vacunas.
- No hay validación de formato en la base para `DNI` y `TELEFONO`.

---

## 9. Pendientes conocidos

- Unicidad por `DETALLE_CITA` (atención y cobro) solo en el backend. Si se quiere garantía en la base, se puede añadir una columna generada que valga `NULL` cuando el registro está anulado y un UNIQUE sobre ella. Es sencillo para `ATENCION_MEDICA`, pero para `DETALLE_BOLETA_SERV` habría que copiar `ANULADO` de la boleta.
- Charset del esquema: `utf8` (utf8mb3). Conviene `utf8mb4`.
- `HORARIO.DIA_SEMANA` sin catálogo ni validación de horas.
- `ARTICULO` sin tipo (medicamento, insumo, producto de venta).
- `CUENTA` sin rol explícito.
- Sin índices pensados para reportes por fecha (`CITA.FECHA_HORA`, `BOLETA.FECHA`).
- `DETALLE_BOLETA_SERV.CANTIDAD` es redundante: siempre vale 1.
- El diagrama `diagrama-fisico_patitas.mwb` debe actualizarse con los últimos cambios (`SUBTOTAL` generado, `ANULADO` en `ATENCION_MEDICA`, `STOCK_ACTUAL UNSIGNED`, `SEXO` enum, UNIQUE de `DETALLE_CITA (ID_CITA, ID_SERVICIO)`, sin UNIQUE en `ID_DETALLE_CITA` de atención y boleta, `DETALLE_BOLETA_SERV` con `ID_DETALLE_CITA`, y el enum de `SERVICIO` sin `EMERGENCIA`).
- El script no se ha probado contra un servidor MySQL.
