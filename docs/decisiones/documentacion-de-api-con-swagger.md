# Documentación de API con Swagger: dónde vive el contrato y cómo se expone

**Fecha:** 17/09/2026
**Tarea relacionada:** T&lt;N&gt;

## Contexto

El backend no tenía ningún lugar donde consultar el contrato de la API. El skill
`backend-express-capas` cierra con "actualizar la documentación de la API (donde esté
centralizada en el repo)", pero ese lugar no existía: el único registro de qué hace cada endpoint
era el `CHANGELOG.md`, que es un historial de cambios y no un contrato. Para saber qué acepta
`POST /patients` había que leer el schema Zod, el controlador y el serializer.

Al mismo tiempo, `nutria-ia` sí documenta solo: FastAPI genera su OpenAPI sin trabajo adicional.
Eso deja dos mitades del sistema documentadas de formas distintas y en lugares distintos, y para
ver el contrato completo hay que abrir dos pestañas y acordarse de dos dominios.

Hay entonces dos preguntas encadenadas: dónde vive el contrato del backend, y cómo se consultan
los dos servicios juntos.

## Alternativas consideradas

**Dónde vive el contrato**

- **Seguir con el `CHANGELOG.md`.** Cero trabajo. En contra: no es consultable (hay que leer
  todas las entradas para reconstruir el estado actual de un endpoint), no es probable, y no
  distingue lo que la API hace hoy de lo que cambió en la iteración 3.
- **Un `docs/api.md` escrito a mano.** Consultable. En contra: nada obliga a actualizarlo, nada
  verifica que sea cierto, y se desincroniza en la primera semana.
- **Spec OpenAPI generado desde comentarios en las rutas, con `swagger-jsdoc`.** El contrato vive
  al lado del código que lo implementa, y se puede testear. En contra: los schemas de componentes
  se escriben a mano y hay que acordarse de tocarlos junto con el schema Zod.
- **Derivar los schemas de Zod con `z.toJSONSchema`.** Una sola fuente de verdad. En contra: Zod 4
  no puede representar el `superRefine` de "menor de edad exige adulto responsable" —
  precisamente la regla más difícil de descubrir leyendo el código — y falla o emite `{}` sobre
  el `preprocess` + `transform` de `birthDate`. Además el spec necesita descripciones y ejemplos
  que en Zod no existen, así que tampoco elimina el trabajo manual.

**Cómo se exponen los dos servicios**

- **Dos UIs separadas.** Es el estado actual. En contra: no resuelve nada.
- **Fusionar los dos specs en un documento único.** Una sola definición para todo. En contra:
  `/health` existe en los dos servicios con distinto `service` y colisiona; obliga a un paso de
  build que se desincroniza; y hace que el backend sea dueño de un contrato que no implementa.
- **Una UI con dos definiciones en el selector nativo de Swagger UI.** Cada servicio sigue siendo
  dueño de su spec y no hay nada que sincronizar. En contra: hay que resolver que el navegador
  pueda leer el spec de IA y ejecutar requests contra ella desde otro origen.

**Cómo llega el navegador al servicio de IA**

- **Agregarle CORS a `nutria-ia`** para que acepte el origen de la documentación. Son seis líneas.
  En contra: a ese servicio no lo llama ningún navegador — el frontend va siempre por el backend —
  así que declarar orígenes permitidos describe algo que no pasa, y ensucia su `.env.example` con
  una variable que existe solo por una herramienta de otro repo. Encima obliga a la variable
  `IA_PUBLIC_URL` en el backend y a sumar el dominio de la documentación al `CORS_ORIGIN`
  del backend. Tres piezas de config repartidas en dos repos para sostener un botón.
- **Que todo salga contra el mismo origen que sirve la documentación**: `servers` relativos, y el
  backend reenvía las llamadas de IA igual que ya reenvía su spec. En contra: ~25 líneas de
  reenvío en el backend.

## Decisión

El contrato de la API del backend pasa a estar centralizado en un spec OpenAPI generado con
`swagger-jsdoc` desde los bloques `@openapi` de `src/routes/*.routes.js`, con los componentes
reutilizables escritos a mano en `src/docs/components.js`. Eso es lo que el paso de docs del
skill `backend-express-capas` pide actualizar al agregar un endpoint. El `CHANGELOG.md` sigue
siendo el registro de qué cambió en cada iteración, no el contrato.

Los dos servicios se consultan en una sola UI (`http://swagger.local.nutria.com`, servida por el
contenedor del backend y no montada cuando `NODE_ENV=production`), que carga **dos definiciones
separadas** en el selector de Swagger UI en vez de fusionarlas.

Todo lo que la documentación dispara sale contra el mismo origen que la sirve: los `servers` del
spec del backend arrancan con una URL relativa, y el spec de `nutria-ia` lo pide el backend por la
red interna de Docker y lo reexpone en `/docs/ia.json` con un `servers` que apunta a un reenvío
propio (`/docs/ia/*`), también en el backend. Nada cruza orígenes, así que **no hay CORS en ningún
lado de esta solución**: `nutria-ia` no se entera de que la documentación existe.

## Consecuencias

El contrato es ahora consultable, probable y ejecutable, y vive al lado del código que lo
implementa. Agregar un endpoint sin documentarlo hace fallar un test, porque la lista de
operaciones esperadas está escrita literalmente en `src/docs/openapi.test.js`. Eso es deliberado:
las fallas de `swagger-jsdoc` son silenciosas y un bloque YAML mal indentado borra la operación
del spec sin error ni advertencia.

El costo asumido es la duplicación entre el schema Zod y el schema OpenAPI del mismo endpoint.
Se paga a cambio de poder documentar lo que Zod no expresa (la regla del adulto responsable) y lo
que Zod no tiene (descripciones, ejemplos, la forma de las respuestas). Si más adelante el
esquema crece mucho, la alternativa a revisar no es generar desde Zod sino agregar un test que
compare ambos, no reemplazar uno por el otro.

Documentar la API tal como es, y no como convendría que fuera, dejó asentadas tres
inconsistencias reales que antes no se veían: el `404` de ruta inexistente usa `{ error, path }`
en vez del `{ error, fields? }` del resto de la API; el `400` de login no trae `fields` mientras
que el de pacientes sí; y `birthDate` entra como `AAAA-MM-DD` y sale como fecha-hora ISO.
Ninguna se corrigió en esta tarea — son cambios de contrato — pero quedan visibles para decidirlo
en una propia.

El costo del reenvío lo paga el repo que necesita la documentación, y no se reparte sobre un
servicio que no tiene nada que ver con ella. `nutria-ia` queda sin una sola línea agregada por
esta tarea: no sabe que esta documentación existe y no le hace falta saberlo. Ese es el criterio
que ordenó la decisión, y el que conviene aplicar si más adelante aparece la tentación de meter
config de una herramienta dentro de un servicio.

Que el reenvío viva bajo `/docs/` no es cosmético: hereda el gating por entorno de todo el router
de documentación, así que en un entorno desplegado no existe. Si en algún momento se lo mueve
fuera de ahí, deja de ser una herramienta de desarrollo y pasa a ser una vía de acceso al servicio
de IA sin autenticación — hay un test que lo verifica para que el cambio no pase inadvertido.

Este camino es además el único que sobrevive al Incremento 3, donde el servicio de IA se despliega
privado con autenticación por identity token: el navegador no lo alcanza de ninguna forma y todo
tiene que pasar por el backend igual. Cuando llegue ese momento, el reenvío va a necesitar
adjuntar el identity token, que es exactamente lo que el backend ya va a estar haciendo para sus
llamadas reales.
