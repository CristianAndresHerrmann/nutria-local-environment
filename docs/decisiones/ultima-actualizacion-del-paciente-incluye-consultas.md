# La última actualización del paciente incluye sus consultas

**Fecha:** 19/10/2026
**Tarea relacionada:** T19

## Contexto

T19 pide que, al guardar una consulta, la ficha refleje la nueva "última actualización". Ese dato
sale de `patient.updated_at` (ver [ajustes al modelo del Anexo A](ajustes-al-modelo-del-anexo-a.md)),
que también ordena el listado de pacientes. Prisma lo mueve solo cuando se escribe la fila del
paciente. La consulta es otra tabla y tiene su propio `updated_at`, así que registrarla no cambia
el del paciente.

## Alternativas consideradas

- **Calcular la última actualización al leer**, como el máximo entre `patient.updated_at` y el
  `updated_at` de sus consultas. No escribe nada de más. En contra: el listado tendría que sumar
  ese cálculo por paciente para mostrar el dato y ordenar por él, y cada tabla nueva del paciente
  (mediciones en T20, planes) tendría que agregarse al cálculo.
- **Mover `patient.updated_at` al guardar la consulta**, en la misma transacción. El listado y la
  ficha siguen leyendo una sola columna. En contra: la columna deja de significar "se editó la fila
  del paciente" y pasa a significar "se registró algo del paciente".

## Decisión

Guardar o editar una consulta actualiza `patient.updated_at` en la misma transacción que la
consulta (`consultation.repository`). El `updateMany` filtra por el paciente y por su
nutricionista, como el resto de las escrituras. Si la consulta no existe o es ajena, no se toca
al paciente.

## Consecuencias

- La ficha y el listado muestran la última actualización sin cambios en `patient.repository`.
  El listado además sube al paciente que acaba de tener una consulta.
- Las condiciones del paciente (T18) hoy no mueven `patient.updated_at`. Si se quiere el mismo
  criterio para ellas, hay que tocar al paciente en su repositorio.
- Desde T20, las mediciones siguen el mismo criterio. Darlas de alta, editarlas o borrarlas toca
  al paciente en la misma transacción, también cuando se cargan con la consulta. La función
  `touchPatient` pasó a `src/repositories/touchPatient.js` y la comparten los dos repositorios.
