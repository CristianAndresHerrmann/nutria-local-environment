# Rutas de la API en inglés (`/patients`, no `/pacientes`)

**Fecha:** 16/09/2026
**Tarea relacionada:** T6

## Contexto

T5 expuso `GET /patients` y `GET /patients/:id`. El enunciado de T6 nombra los
endpoints como `/pacientes`. Antes de agregar `POST` y `PUT` hay que definir cuál
de las dos formas es el contrato real, porque el frontend se escribe contra ella y
cambiarla más tarde obliga a tocar los dos repos a la vez.

## Alternativas consideradas

- **Castellano (`/pacientes`)** — coincide literalmente con el enunciado de la
  tarea y con el vocabulario del dominio. En contra: obliga a renombrar lo que ya
  entrega T5 antes de mergearlo, y deja la API en un idioma distinto al de los
  identificadores del modelo (`Patient`, `nutritionistId`).
- **Inglés (`/patients`)** — consistente con el código que ya existe y con la
  convención del `schema.prisma` (identificadores en inglés, datos en español).
  En contra: el nombre del endpoint no calca el texto del enunciado.

## Decisión

La API se expone en inglés: `/patients`. El `/pacientes` del enunciado se
interpreta como nombre funcional de la pantalla, no como contrato literal de la
ruta. Las URLs del frontend (`/pacientes`, `/pacientes/:id`) sí quedan en
castellano, porque son visibles para el usuario.

## Consecuencias

Se mantiene un solo idioma para todo lo que es identificador técnico (modelo
Prisma, rutas, nombres de campos del JSON) y otro para lo que ve el
nutricionista (datos, textos de UI, URLs del navegador). No hay que retocar T5
antes de mergearlo. Si en el futuro se documenta la API para un tercero, el
enunciado de la tarea y la ruta real no coinciden textualmente: queda asentado
acá el porqué.
