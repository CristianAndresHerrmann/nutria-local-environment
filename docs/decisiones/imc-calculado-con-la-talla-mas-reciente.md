# IMC calculado al leer, con la talla más reciente a la fecha del peso

**Fecha:** 20/10/2026
**Tarea relacionada:** T20

## Contexto

RF-11 pide mostrar el IMC cuando hay peso y talla disponibles. El Anexo A no lo incluye en
MEDICION: es un dato derivado. Peso y talla son filas independientes del histórico y pueden
cargarse en fechas distintas. Un adulto rara vez vuelve a medirse la talla, y el peso cambia en
cada consulta. Hay que decidir dónde se calcula el IMC, con qué talla y en qué unidades.

## Alternativas consideradas

- **Guardar el IMC como una medición más**, al registrar un peso. La lectura es directa. En
  contra: es un dato derivado que se desincroniza si se corrige el peso o la talla, y duplica lo
  que ya está en la tabla.
- **Calcularlo en el frontend** con el histórico que recibe. No toca la API. En contra: la regla
  queda fuera del backend y habría que repetirla en el PDF o el plan (Incrementos 3 y 4).
- **Calcularlo en el backend al leer**, con la talla más reciente de fecha igual o anterior a la
  del peso. Siempre refleja el histórico vigente y la regla vive en un solo lugar. En contra: cada
  respuesta tiene que leer las tallas del paciente.
- **Usar la última talla cargada, sin mirar la fecha.** Más simple. En contra: un peso viejo
  quedaría con una talla tomada después, que en ese momento no se conocía. Para un menor en
  crecimiento, eso cambia el valor.

## Decisión

El IMC se calcula en el backend al leer (`src/utils/bmi.js`) y no se guarda. Para cada peso se
usa la talla más reciente con fecha igual o anterior a la del peso. Sin esa talla, `bmi` es
`null`. Viaja en cada medición de `GET /patients/:id/measurements` (`null` en los tipos que no son
peso) y en la respuesta del alta y la edición.

- **Unidades fijas.** El peso va en kg y la talla en cm. Si no se manda la unidad, el schema la
  completa. Otra unidad responde `400` en vez de convertirse, para no guardar un valor que el
  profesional no escribió. El mismo criterio se extendió a los demás tipos, para que el histórico
  no mezcle unidades:
  - Pliegue en mm, circunferencia en cm, y colesterol, triglicéridos y glucemia en mg/dL.
  - `other` elige su unidad de una lista de unidades de laboratorio, o va sin unidad.
  - El frontend ofrece la unidad en un desplegable según el tipo.
- **Redondeo.** El IMC tiene un decimal.
- **Sin clasificación.** No hay categorías ni umbrales: serían una regla clínica, y no está en el
  alcance de T20.
- **Dos tallas del mismo día.** `measurement` no tiene fecha de alta, así que se desempatan por id.
  Es un orden arbitrario pero estable: el resultado no cambia entre lecturas. No se agregó una
  migración para un caso que en la práctica se resuelve editando o borrando la talla repetida.

## Consecuencias

- Corregir un peso o una talla recalcula solo los IMC afectados.
- El listado ordena las mediciones del mismo día por tipo y por id, también por la falta de fecha
  de alta. Si algún día hace falta el orden de carga, se agrega `created_at` con una migración y se
  cambia el desempate.
- Los gráficos de evolución y la variación entre consultas (RF-19 a RF-21, Incremento 5) pueden
  reusar `withBmi` sobre el mismo histórico.
