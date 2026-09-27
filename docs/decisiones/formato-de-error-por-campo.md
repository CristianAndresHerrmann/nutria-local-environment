# Formato de error con detalle por campo

**Fecha:** 16/09/2026
**Tarea relacionada:** T6
**Estado:** reemplazada por [formato-de-error-uniforme.md](formato-de-error-uniforme.md) (T7).

## Contexto

El formato de error uniforme que dejó T5 es `{ "error": "mensaje" }`: un único
string. El alta/edición de pacientes de T6 tiene validaciones cruzadas (un
paciente menor de 18 exige los datos del adulto responsable) y el frontend debe
mostrar cada mensaje junto al input que falló, sin inventar textos distintos a los
que devuelve el servidor. Con un string suelto, el frontend no puede saber a qué
campo corresponde el error.

## Alternativas consideradas

- **Dejar `{ error }` y parsear el texto en el frontend** — no cambia nada del
  backend. En contra: acopla la UI a la redacción exacta del mensaje y se rompe
  ante cualquier cambio de wording.
- **Reemplazar por `{ errors: [...] }`** — más expresivo. En contra: rompe el
  contrato que T5 ya entrega y obliga a tocar el manejo de errores de login.
- **Extender a `{ error, fields? }`** — agrega el detalle sin quitar nada.

## Decisión

El formato pasa a ser `{ "error": "mensaje", "fields"?: { "campo": "mensaje" } }`.
`error` sigue presente en toda respuesta de error, con el mismo significado que en
T5. `fields` es opcional y solo aparece en errores de validación (400), mapeando
nombre de campo a su mensaje.

## Consecuencias

Nada de lo que consume la API hoy se rompe: quien solo lee `error` sigue
funcionando igual. El frontend puede pintar errores por campo usando exactamente
el texto del servidor. El costo es que el `errorHandler` y los controladores que
validan con Zod tienen que armar el mapa de campos a partir de
`error.issues[].path`, en vez de tomar solo el primer mensaje.
