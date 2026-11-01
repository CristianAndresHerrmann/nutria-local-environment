# Medición de la generación con un script contra el stack local

**Fecha:** 01/11/2026
**Tarea relacionada:** T41

## Contexto

La Tabla 7 del Informe de Avances 2 pide una medición preliminar en el Incremento 3 de
dos metas:

- El tiempo desde la solicitud hasta la propuesta validada, en P95, con meta de 15 s.
- La proporción de propuestas sin hallazgos bloqueantes, con meta del 95 %.

DA-12 necesita además comparar proveedores en tokens y costo.
[`estrategia-de-pruebas.md`](estrategia-de-pruebas.md) había previsto el tiempo como una
prueba de carga con k6, sobre el stack aislado del E2E, y el 95 % como una evaluación del
modelo.

Hubo que resolver lo siguiente:

- **De dónde salen la duración del proveedor y los tokens.** La API no los expone: el plan
  trae solo el resumen de la generación (proveedor, modelo, plantilla y resultado de la
  validación).
- **Qué se registra de una generación fallida.** El `503` trae solo el `reason`, y los
  tokens de una salida inválida igual se cobran.
- **Dónde se corre.** El stack aislado del E2E todavía no existe, y la tarea pide correr
  contra el stack local.
- **Con qué concurrencia.** Cada generación con un proveedor real tiene costo y está
  sujeta al límite de solicitudes.

## Alternativas consideradas

- **k6, como prevé la estrategia.** Trae los percentiles y los umbrales, y los escenarios
  servirían para la carga del Incremento 4. En contra: solo ve HTTP. Para tener la
  duración del proveedor y los tokens habría que agregarlos a la respuesta del plan, un
  cambio de API sin otro uso. Y de una falla no tendría más que el `reason`. Tampoco puede
  reusar los casos del seed, que están en un módulo CommonJS.
- **Exponer la traza completa en la API** (`durationMs`, tokens). Le serviría a k6 y a un
  futuro panel. En contra: el editor no la usa, y la traza es trazabilidad (RF-41), no
  parte del plan.
- **Medir desde el navegador** (DevTools), como en las muestras del 01/11. Da el tiempo
  real del editor, pero a mano, de a una generación y sin hallazgos agregados.
- **Concurrencia de varios usuarios.** Es lo que mide una prueba de carga. Para la
  generación multiplica el costo, y mezcla la latencia del proveedor con su límite de
  solicitudes.

## Decisión

Un script Node en `nutria-backend/scripts/measure-generation.js`, lanzado a mano con
`npm run measure:generation` desde la raíz:

- **Dónde corre:** en un contenedor del backend, dentro de la red del compose, contra el
  stack local levantado.
- **Cómo mide:** inicia sesión con la cuenta del seed donde están los casos clínicos de
  `scripts/seed-data/clinical-cases.js`. Corre N generaciones por caso, en serie y en
  round-robin. El tiempo total se mide alrededor del `POST`, y la duración del proveedor y
  los tokens se leen con Prisma de la traza `ai_generation`, también para las fallas que
  llegaron al proveedor.
- **Costo:** sale de una tabla de precios en el script, con la fuente y la fecha de cada
  precio.
- **Salida:** un JSON por corrida en `docs/mediciones-de-generacion/` del repo raíz, y las
  tablas en Markdown para el README de esa carpeta.
- **Pruebas:** las funciones de cálculo (percentil, resumen, costo, búsqueda de casos)
  tienen tests unitarios en la suite del backend.
- **Proveedor:** el que tenga configurado `nutria-ia`. Para comparar se cambia el `.env` y
  se recrea el servicio, el mismo procedimiento de DA-12.

## Consecuencias

- **La medición es preliminar.** Es secuencial y en el stack local, con Docker Desktop. La
  definitiva del Incremento 4 se repite con el proveedor elegido, y en staging si se quiere
  incluir la red real.
- **No reemplaza la prueba de carga.** RNF-01, RNF-02 y RNF-11 con concurrencia siguen
  previstos con k6 en [`estrategia-de-pruebas.md`](estrategia-de-pruebas.md). Si el k6
  necesita tokens o duración, se decide entonces si exponerlos.
- **Es un script versionado, no una prueba.** No tiene aserciones sobre el sistema ni
  corre en la CI, y está justificado porque mide con costo y a demanda. La lógica que puede
  romperse tiene tests, y la presencia de los casos la verifica la integración del seed.
- **Los borradores generados quedan en la base local** de la cuenta de los casos, a la
  vista en el editor. `npm run db:reset` los borra.
- **El seed exporta la contraseña de las cuentas**, y su `main` corre solo cuando se lo
  ejecuta directo.
- **El nivel de razonamiento no queda en la traza.** Se anota a mano con `--note` y en el
  README de la corrida.

## Ampliaciones de la misma tarea

Las mediciones mostraron que el tiempo depende de los tokens de salida, y para seguir el
riesgo 10 se sumaron dos cosas:

- **Los tokens de razonamiento, aparte en la traza.** `nutria-ia` los informa en
  `generation.reasoning_tokens`: en OpenAI, `output_tokens_details.reasoning_tokens`; en
  Gemini, `thoughts_token_count`. Son una parte de `output_tokens`, que no cambia.
  - El backend los guarda en `ai_generation.reasoning_tokens`, una columna nullable con
    el mismo CHECK de métricas no negativas (migración
    `agrega_tokens_de_razonamiento_a_generacion`).
  - `null` significa que el proveedor no los informó, no que valgan cero. Gemini no los
    informó en ninguna corrida.
  - Por qué una columna y no derivarlo de la respuesta cruda: el razonamiento no está en
    la respuesta, solo en el uso que informa la API.
- **La plantilla de prompt, seleccionable.** `AI_PLAN_TEMPLATE` elige entre las plantillas
  publicadas, con `plan_proposal_v1` por defecto. Una que no existe impide arrancar el
  servicio. La traza registra la versión usada, así que las corridas de distintas
  plantillas quedan distinguidas en `ai_generation` y en el JSON de la medición.
  - Se publicó `plan_proposal_v2`, con salida más corta, para medir. No es la vigente:
    adoptarla es una decisión de contenido, para la entrevista.
  - Alternativa descartada: medir la v2 con un contenedor de `nutria-ia` aparte y la
    plantilla montada. No hacía falta tocar el código, pero la traza habría registrado
    la versión equivocada, y no se podía medir por el backend.
