# Análisis del campo `ACTIVO` en `Patitas_ArribaDDL_V2.sql`

## Criterio: cuándo una tabla necesita `ACTIVO`

`ACTIVO` (borrado lógico) solo sirve cuando se cumplen estas dos condiciones:

1. **Otros registros históricos la referencian**, así que no se puede hacer `DELETE` sin romper una FK ni perder historia.
2. **Existe la acción de "desactivar"** como operación propia (dar de baja, descontinuar, anular).

Las tablas de detalle e intermedias casi nunca cumplen eso. Su ciclo de vida depende del padre. Si se anula la boleta, se anula la boleta entera, no línea por línea. Y mientras el padre es editable, quitar una línea es un `DELETE` normal.

## Veredicto por tabla (según V2)

| Tabla | V2 hoy | Recomendación | Motivo |
|---|---|---|---|
| CUENTA | tiene | **Mantener** | Permite bloquear el login sin borrar nada |
| CLIENTE | tiene | **Mantener** | Lo referencian mascotas y boletas |
| ADMINISTRADOR | tiene | **Mantener** | Se da de baja a una persona, no se borra |
| VETERINARIO | tiene | **Mantener** | Lo referencian citas históricas |
| MASCOTA | tiene | **Mantener** | Fallecida o transferida, pero con historial clínico |
| ARTICULO | tiene | **Mantener** | Se descontinúa y las boletas viejas lo referencian |
| SERVICIO | tiene | **Mantener** | Mismo caso que ARTICULO |
| CATEGORIA | tiene | **Mantener** (opcional) | Catálogo, y no cuesta nada |
| DIAGNOSTICO | tiene | **Mantener** | Catálogo referenciado por atenciones |
| HORARIO | tiene | **Opcional** | Mantener solo si los horarios son franjas compartidas que "se retiran" |
| BOLETA | tiene | **Mantener, con otro significado** | Aquí `ACTIVO = anulada`. Es un documento de venta que no se borra |
| RECETA | tiene | **Mantener** | Es registro clínico y no debería borrarse |
| ATENCION_MEDICA | tiene | **Mantener** | Mismo caso: registro clínico |
| **DETALLE_RECETA** | tiene | **Quitar** | Es detalle puro. Si se anula la receta, se anula todo |
| **HORARIO_ADMINISTRADOR** | tiene | **Quitar** | Nada referencia la asignación, así que se puede borrar físicamente |
| **HORARIO_VETERINARIO** | tiene | **Quitar** | Mismo caso. Solo se mantendría si se quiere historial de horarios |
| DETALLE_BOLETA_ART / SERV | no tiene | **Dejar sin él** | Decisión correcta |
| CITA | no tiene | **Dejar sin él** | `ESTADO` ya cumple esa función |
| DETALLE_CITA | no tiene | **Dejar sin él** | Se edita o borra mientras la cita está pendiente |
| ATENCION_DIAGNOSTICO | no tiene | **Dejar sin él** | Tabla intermedia |
| INSUMO_UTILIZADO | no tiene | **Dejar sin él** | Es registro de consumo |
| CATEGORIA_ARTICULO | no tiene | **Dejar sin él** | Tabla intermedia |

## Hallazgos

1. **V2 es inconsistente con su propio criterio.** `DETALLE_RECETA` y las dos tablas `HORARIO_*` aún tienen `ACTIVO`, mientras que `DETALLE_BOLETA_*` ya no. Hay que decidir una sola regla y aplicarla en todo el modelo.
2. **`CITA` sin `ACTIVO` está bien, pero `ESTADO` tiene que cubrirlo todo.** Hoy tiene `PENDIENTE, APROBADO, RECHAZADO, CANCELADO`. Cancelar es una baja lógica, así que no hace falta `ACTIVO`. Pero falta un estado final como `ATENDIDO` o `COMPLETADO`, y no queda claro qué hace la columna `PROGRAMADO`.
3. **`CUENTA.ACTIVO` y `CLIENTE.ACTIVO` se solapan** (igual para administrador y veterinario). Hay que definir cuál manda. Sugerencia: `CUENTA.ACTIVO` controla el acceso al sistema, y el `ACTIVO` de la persona controla si aparece en listados, agendas y citas nuevas.
4. **`BOLETA.ACTIVO` es ambiguo por el nombre.** "Activo" suena a vigente, pero lo que se quiere expresar es "anulada". Si se van a enumerar estados, podría ser un `ESTADO` en lugar de un booleano.
5. **Quitar `TRATAMIENTO` y `ATENCION_TRATAMIENTO` tiene sentido** si el tratamiento realizado es el servicio de `DETALLE_CITA`. Dos consecuencias:
   - `INSUMO_UTILIZADO.ID_ATENCION_TRATAMIENTO` debe eliminarse. Ahora es una columna NOT NULL sin FK y sin tabla destino.
   - Un insumo debería colgar de un **solo** padre. Hoy lo hace de `ATENCION_MEDICA`, pero `ATENCION_MEDICA` está ligada a un solo `DETALLE_CITA`. Si una cita tiene 3 servicios, se necesitarían 3 atenciones médicas. Hay que confirmar si eso es lo deseado o si una cita debe tener una única atención con varios servicios. Eso cambia dónde va la FK.
6. **Riesgo de quitar `ACTIVO` en los detalles:** un `DELETE` sobre `DETALLE_CITA` falla si ya existe una `ATENCION_MEDICA` que lo referencia. Esto protege el historial clínico, pero la aplicación debe manejar ese error.

## Regla propuesta

> `ACTIVO` solo en **entidades maestras** (cuentas, personas, mascotas, catálogos) y en **documentos que no se borran** (boleta, receta, atención médica). Nada en detalles, tablas intermedias ni registros de consumo. Los flujos de estado (citas) usan `ESTADO`.

Con esa regla, a V2 le sobra `ACTIVO` en `DETALLE_RECETA`, `HORARIO_ADMINISTRADOR` y `HORARIO_VETERINARIO`.

## Decisiones pendientes

1. ¿Se quita `ACTIVO` de `DETALLE_RECETA`, `HORARIO_ADMINISTRADOR` y `HORARIO_VETERINARIO`, o se quiere mantener historial de horarios?
2. ¿Una cita con varios servicios genera una sola atención médica o una por servicio?
3. ¿`BOLETA.ACTIVO` se mantiene como booleano de anulación o pasa a un `ESTADO`?
4. ¿Qué manda en el solapamiento entre `CUENTA.ACTIVO` y el `ACTIVO` de cliente/administrador/veterinario?

## Siguientes pasos

- Aplicar las decisiones anteriores en V2.
- Corregir los enums (`METODO_PAGO`, `TIPO_SERVICIO_MEDICO`, `ESTADO` de cita).
- Corregir los `DECIMAL` sin precisión (`TOTAL`, `PRECIO_BASE`, `SUBTOTAL`, `PESO_FISICO`).
- Eliminar `INSUMO_UTILIZADO.ID_ATENCION_TRATAMIENTO`.
