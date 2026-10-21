# Condiciones repetidas del paciente controladas en el servicio

**Fecha:** 19/10/2026
**Tarea relacionada:** T18

## Contexto

T18 pide que una condición repetida (mismo tipo y mismo alimento, grupo o patología) responda
`409 CONFLICT`. En `patient_condition`, el destino está en tres columnas excluyentes
(`pathology_id`, `food_id`, `food_group_id`), y cada fila tiene al menos dos de ellas en `NULL`.

## Alternativas consideradas

- **`@@unique([patientId, type, pathologyId, foodId, foodGroupId])` en Prisma.** Es lo que
  Prisma sabe expresar. En contra: Postgres trata los `NULL` como distintos en un índice único,
  así que nunca dispararía.
- **Índice único con `NULLS NOT DISTINCT`, o tres índices parciales, escritos a mano en el
  SQL.** La base garantizaría la unicidad. En contra: Prisma 6 no los expresa en
  `schema.prisma`, así que los detecta como drift y propone borrarlos en la migración
  siguiente. Es el mismo problema que descartó la FK compuesta en
  [ajustes al modelo del Incremento 2](ajustes-al-modelo-del-anexo-a-incremento-2.md).
- **Chequeo en el servicio, sin restricción en la base.** Es el mismo criterio que ya usa la
  unicidad de nombres de los alimentos comunes (T15). En contra: dos requests simultáneas
  iguales podrían pasar las dos.

## Decisión

El servicio busca una condición del mismo paciente con el mismo tipo y destino antes del alta y
de la edición (al editar, excluye la propia) y responde `409` con el campo del destino en
`details.fields`. No hay índice único.

## Consecuencias

- No hace falta una migración, y no hay nada escrito a mano que Prisma pueda intentar borrar.
- La carrera entre dos altas simultáneas iguales queda como riesgo asumido. El único usuario es
  el nutricionista desde su propia sesión, y el frontend deshabilita el botón mientras guarda.
  En el peor caso, una condición queda repetida y se borra desde la ficha.
- Si una versión futura de Prisma soporta índices parciales o `NULLS NOT DISTINCT` sin drift,
  conviene sumar la restricción en la base y dejar el chequeo del servicio para el mensaje por
  campo.
