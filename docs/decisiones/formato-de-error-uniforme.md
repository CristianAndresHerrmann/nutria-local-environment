# Formato de error uniforme `{ error: { code, message, details? } }`

**Fecha:** 23/09/2026
**Tarea relacionada:** T7

## Contexto

T6 dejó el formato `{ error: "mensaje", fields?: {...} }` (ver
`formato-de-error-por-campo.md`). Al documentar la API con Swagger quedaron a la vista
tres inconsistencias: el `404` de ruta inexistente usaba `{ error, path }`, el `400` de
login no traía `fields` y el de pacientes sí, y cualquier error no previsto (un error
de Prisma, el `SyntaxError` de un body mal formado) salía con su `message` interno
tal cual.

T7 pide un manejador centralizado con formato `{ error: { code, message, details? } }`.
Además, el frontend tiene que distinguir casos que comparten status sin parsear el
texto del mensaje: validación contra JSON mal formado (los dos son 400), o el servicio
de IA caído contra la base caída (los dos son 503).

## Alternativas consideradas

- **Mantener `{ error, fields? }` y agregar `code` al lado.** No rompe nada. En contra:
  `error` sigue siendo un string en el nivel raíz, así que no queda lugar para detalles
  que no sean de un campo (el `reason` de la IA caída) sin agregar más claves sueltas.
- **`details` como mapa campo → mensaje directo.** Más corto. En contra: ata `details`
  a la validación, y cualquier otro detalle se confundiría con un nombre de campo.
- **`{ error: { code, message, details? } }` con los campos en `details.fields`.**
  Rompe el contrato de T6, pero la API todavía no tiene más consumidor que el frontend
  del mismo proyecto.

## Decisión

Toda respuesta de error de la API, incluido el `404` de ruta inexistente, es
`{ "error": { "code": "...", "message": "...", "details"?: { ... } } }`:

- `code` es estable y en mayúsculas (`VALIDATION_ERROR`, `INVALID_JSON`,
  `UNAUTHORIZED`, `NOT_FOUND`, `CONFLICT`, `INTERNAL_ERROR`, `IA_UNAVAILABLE`, …). Si
  no se indica, sale del status.
- `message` está en castellano y se puede mostrar al usuario tal cual.
- `details` es un objeto abierto y opcional. En validación trae `fields`, un mapa de
  campo → mensaje; en `IA_UNAVAILABLE` trae `reason`.

Un error que el `errorHandler` no reconoce responde `500 INTERNAL_ERROR` con un
mensaje fijo. El `message` original y el stack van solo al log del servidor.

## Consecuencias

El frontend decide qué hacer por `code`, muestra `message` sin traducirlo y pinta
`details.fields` junto a cada input. Toda esa lectura vive en un solo lugar, el
cliente HTTP (`apiRequest`), que además convierte los errores de red y de timeout al
mismo tipo `ApiError`.

Es un cambio de contrato: cualquier consumidor que leía `error` como string deja de
funcionar. Se asume ahora porque el único consumidor es el frontend propio, que se
adapta en la misma tarea; más adelante el costo sería mayor.

Validar con Zod pasa del controlador a un middleware `validate` en la ruta, que arma
`details.fields` igual para todos los endpoints. El login ahora también devuelve
errores por campo. Queda una excepción deliberada: en `/patients/:id` un id que no es
UUID sigue respondiendo `404` sin `details`, para que no se distinga de un paciente
inexistente o de otro nutricionista.
