# Estrategia de pruebas: un tipo de prueba por objetivo

**Fecha:** 24/10/2026
**Tarea relacionada:** T23

## Contexto

Hasta el Incremento 2, el proyecto tenía solo pruebas unitarias. En el backend usan
repositorios fake, más una integración liviana de la app sin base (`src/app.test.js`). En
el frontend cubren funciones puras. La regla era sumar integración y E2E "de forma
selectiva".

Al cerrar el Incremento 2 (T23) hubo tres cosas que ninguna prueba cubría:

- El aislamiento por nutricionista contra Postgres real, los `CHECK` de las migraciones
  y las transacciones.
- El recorrido por las pantallas y los errores de validación junto a cada campo.
- El tiempo de carga de la ficha (RNF-01).

Se verificaron con scripts descartables y a mano. No se quieren scripts sueltos sin un
propósito explícito en los repos: lo que verifica el sistema tiene que ser una prueba,
con su comando y su momento de ejecución.

## Alternativas consideradas

- **Seguir con unitarias y verificación manual al cierre de cada incremento.** No suma
  nada que mantener. En contra: la verificación depende de que alguien la repita a mano,
  y lo que valida no queda en ninguna suite.
- **Versionar los scripts de verificación** (recorrido de la API, seed de volumen,
  medición de RNF-01). Reusa lo que ya existe. En contra: no son pruebas. No tienen
  aserciones mantenidas ni fixtures propias, y quedan como código de propósito ambiguo.
- **Un tipo de prueba por objetivo**, cada uno con su herramienta, su comando y su
  momento. Es lo que se eligió.

Dentro de esa opción se decidió además:

- **Base de la integración:** una base `nutria_test` dentro del mismo Postgres, en vez de
  un contenedor efímero aparte. Aísla igual de los datos de desarrollo, sin sumar un
  servicio ni esperar a que arranque otro Postgres.
- **Datos del E2E:** un stack aislado, en vez de la base de desarrollo con una cuenta
  propia. La cuenta propia era más simple, pero el resultado dependería de lo que hubiera
  en esa base y además la ensuciaría.
- **Carga:** k6, en vez de extender un script propio. Mide con concurrencia, trae
  umbrales de percentil incorporados, y los mismos escenarios sirven después para la
  fase de estrés.
- **Lint en el backend:** no por ahora. El backend ya tiene una suite amplia y valida
  todo con Zod. Lo que agregaría un linter (nombres mal escritos en caminos sin test) no
  justifica sumarlo todavía. Si aparece un error de ese tipo, se reconsidera.

## Decisión

| Tipo | Qué cubre | Herramienta | Dónde vive | Comando | Cuándo se corre | Estado |
|---|---|---|---|---|---|---|
| Unitarias | Lógica de negocio, schemas, aislamiento con fakes, spec OpenAPI | Vitest | Junto al código, `*.test.js` | `npm run test:backend`, `npm run test:frontend` | Cada cambio | Implementadas |
| Lint | Errores de código del frontend | oxlint (ya en `nutria-frontend`) | Config del repo | `npm run lint:frontend` (previsto) | Cada cambio | Falta el comando en la raíz |
| Integración (backend) | API real contra Postgres, repositorios, restricciones y transacciones | Vitest, con config aparte | Junto al módulo, `*.int.test.js` | `npm run test:backend:integration` | Al cerrar cada tarea del backend y en CI | Implementada (AUD-001) |
| E2E | Camino feliz de los flujos necesarios, en el navegador | Playwright | `nutria-frontend/e2e/` | `npm run test:e2e` (previsto) | Al cerrar cada tarea que toque esos flujos, y al cerrar el incremento | Definida |
| Carga | Umbrales de los RNF medibles, con pocos usuarios | k6 | A definir al implementarla | `npm run test:load` (previsto) | Al cerrar el incremento, y en staging | Definida |
| Estrés | Dónde se degrada el sistema y qué conviene mejorar | k6 (los mismos escenarios) | Ídem | Ídem, con otra configuración | Cuando haya uso real | Más adelante |

Todo corre dentro de Docker y se lanza desde el `package.json` de la raíz, como el resto
de los comandos. Hasta que exista CI (card T12), "cuándo se corre" es una regla de
trabajo. Cuando exista, CI corre las unitarias y el lint en cada PR, y suma integración
y E2E si los tiempos lo permiten.

### Integración del backend

Implementada para AUD-001: aislamiento de paciente, condición, consulta, medición,
plan y receta privada (API y repositorios reales); atomicidad de `createGenerated`,
`replaceDraft` y consulta con hijos; CHECK de `patient_condition` y `ai_generation`,
FK compuestas, RESTRICT, `citext`, `createManyAndReturn` y seed sobre base vacía.
La validación general de respuestas contra Swagger y los demás escenarios de
integración de esta estrategia siguen pendientes.

- **Base:** `nutria_test`, en el Postgres del compose. Cada corrida arranca con
  `prisma migrate reset --force --skip-seed`, así que de paso prueba las migraciones
  desde base vacía.
- **Protección del destino:** se deriva `nutria_test` desde `DATABASE_URL`. Un
  `TEST_DATABASE_URL` explícito solo se acepta para `nutria_test` y schema `public`;
  se valida antes del reset. La limpieza verifica también la base y schema reales.
- **Datos:** fixtures propias con Prisma, no el seed. Cada archivo vacía las tablas antes
  de cada test, y los archivos corren en serie porque comparten la base. Un test aparte
  verifica que el seed corre sobre base vacía.
- **Cómo:** la app real en un puerto efímero, como `app.test.js`, pero con los
  repositorios reales.
- **Separación:** la config unitaria excluye `*.int.test.js`, así que `test:backend`
  sigue sin base y rápido.
- **Qué cubre:**
  - Por recurso: un nutricionista sobre datos de otro → 404, y los datos del otro quedan
    iguales en la base.
  - Los `CHECK` y las FK `RESTRICT`.
  - Las transacciones: la consulta con sus mediciones y hábitos, y el `updatedAt` del
    paciente.
  - Las reglas que dependen de datos guardados: RF-28 con subgrupos, IMC con las tallas.
  - El contrato con Swagger (ver "Pruebas que se suman").

### E2E

- **Stack aislado** bajo un *profile* del compose, que `npm run up` no levanta:
  - `backend-e2e`: la misma imagen, contra una base `nutria_e2e`. Al arrancar hace
    `migrate reset` y corre el seed.
  - `frontend-e2e`: la misma imagen, apuntando a `backend-e2e`.
  - `e2e`: la imagen oficial de Playwright.
- El navegador le habla a `frontend-e2e` y este a `backend-e2e` por la red de Docker,
  sin el proxy ni el archivo `hosts`. No usa puertos del host, así que convive con el
  stack de desarrollo levantado.
- `nutria_e2e` va aparte de `nutria_test`, porque la integración la resetea mientras
  corre.
- Hay que sumar `frontend-e2e` a `allowedHosts` de Vite y a `CORS_ORIGIN` del
  `backend-e2e`. No prueba el ruteo de Traefik, que es infraestructura.
- **Flujos, solo el camino feliz:**
  1. Login.
  2. Alta de paciente con datos clínicos y contextuales.
  3. Condiciones.
  4. Consulta con mediciones y hábitos.
  5. Receta.
  6. Recomendación.

  Se suman flujos a medida que hagan falta.
- Los errores de validación se cubren con las unitarias de cada mapeo y la revisión de
  cada formulario. No se escriben E2E de casos de error.

### Carga y estrés

- **Fase 1, carga (ahora):** comprueba los requerimientos. Pocos usuarios simultáneos
  (alrededor de 5) durante unos minutos. Pasa o falla según el umbral de cada RNF.
- **Fase 2, estrés (más adelante):** busca métricas para mejorar. Concurrencia en rampa
  hasta que aparecen errores, más una corrida larga para ver si se degrada con el tiempo.
  El resultado es un informe con la latencia, los errores, CPU y memoria de cada
  contenedor y las consultas lentas de Postgres, no un pasa/falla.
- **Dónde corre:** sobre el stack aislado del E2E, con el volumen de referencia del
  RNF-01 cargado por las fixtures de la prueba: 100 pacientes ficticios con hasta 20
  consultas cada uno. Primero en local, como medición preliminar. Los números que
  cuentan son los de staging: Cloud Run con una instancia como máximo y arranque en
  frío.

| Requerimiento | Qué se mide | Desde |
|---|---|---|
| RNF-01: ficha completa < 5 s en P95 | Las requests de la ficha completa con varios usuarios. Se vuelve a medir cuando la ficha incluya planes | Ahora |
| RNF-05: aislamiento | Chequeo dentro de la carga: con varios nutricionistas a la vez, cada respuesta trae solo datos propios | Ahora |
| Generación del plan < 15 s en P95 (RNF propuesto en la tabla 7 del Informe de Avances 2) | De punta a punta: la solicitud, el backend, `nutria-ia`, el proveedor y la validación, con casos ficticios. Con pocos usuarios, por el costo por token y el límite de solicitudes del proveedor | Incremento 3 (preliminar), Incremento 4 (definitiva) |
| RNF-02: la interfaz sigue operativa durante la generación | La parte del backend: mientras hay generaciones en curso, las demás requests siguen cumpliendo el RNF-01, y con la IA lenta o caída cada generación termina en un error controlado | Incremento 3 |
| RNF-11: trazabilidad de la generación | Al terminar una corrida con generaciones concurrentes, hay exactamente una traza por generación | Incremento 3 |

Otros dos requerimientos se miden, pero no con carga. RNF-12 (el tamaño del documento) se
verifica con un test sobre el archivo generado, cuando se fije el umbral. La meta del 95%
de propuestas sin hallazgos bloqueantes es una evaluación del modelo (ver abajo).

### Pruebas que se suman

Cada una entra cuando existe lo que prueba.

**Recomendadas:**

| Prueba | Por qué |
|---|---|
| Aislamiento generado a partir del spec (backend) | RNF-05 es la propiedad más crítica, porque son datos de salud. Recorrer todas las rutas de `/docs.json` y probar "sin token → 401" y "recurso de otro nutricionista → 404" cubre solo cada endpoint nuevo. Un `requireAuth` olvidado ya no depende de que alguien escriba el test a mano |
| Contrato con Swagger (backend, en la integración) | Validar cada respuesta real contra su schema del spec. Swagger es un entregable y el frontend depende de esas formas. Hoy solo se verifica que el spec esté completo, no que coincida con lo que responde la API |
| Sin datos personales hacia el proveedor de IA (`nutria-ia`) | Prohibición de AGENTS.md (RNF-07). Un paciente ficticio con marcas reconocibles en sus datos personales, y el test verifica que ninguna llegue en lo que se envía al proveedor |
| Parseo estricto de la respuesta del modelo (`nutria-ia`) | Respuestas simuladas mal formadas (JSON roto, campos de más o de menos, tipos incorrectos) tienen que rechazarse. Siempre con el proveedor mockeado: gratis, determinístico y sin enviar nada afuera |
| Sugerencia de IA contra el motor de reglas (backend + IA) | La IA no decide restricciones. Si sugiere algo que viola una condición registrada (por ejemplo, gluten a un paciente celíaco), el motor determinístico del backend lo tiene que marcar. Es la garantía clínica central del sistema |
| Contrato backend ↔ `nutria-ia` | Los dos servicios cambian por separado. Lo que manda el cliente del backend y lo que espera el modelo Pydantic tienen que coincidir, por ejemplo contra el spec que publica FastAPI |
| Auditoría de dependencias (`npm audit`, `pip-audit`) | Un comando que detecta vulnerabilidades conocidas en las librerías. Tiene peso en un sistema con datos de salud |

**Opcionales:**

| Prueba | Por qué |
|---|---|
| Versionado de prompts (`nutria-ia`) | Un snapshot de cada prompt con su versión: si el texto cambia sin cambiar la versión, falla. Mantiene la trazabilidad de los prompts |
| Trazas sin datos personales (`nutria-ia`) | Lo mismo que la prueba de envío al proveedor, aplicado a lo que se guarda como traza |
| Embeddings (`nutria-ia`) | 384 dimensiones y el mismo vector para el mismo texto. Una dimensión distinta de la columna de pgvector rompe la búsqueda sin un error claro |
| Evaluaciones del modelo real (`nutria-ia`) | Un set fijo de casos ficticios contra el proveedor, corrido a mano y no en cada cambio. Mide la calidad de las propuestas y alimenta la meta del 95% |
| Accesibilidad | Con axe dentro de Playwright, sobre las pantallas de los flujos E2E |
| Migraciones con datos | Cuando una migración transforme datos existentes, probarla sobre una base con datos |

**No por ahora:** regresión visual (el diseño todavía cambia en Figma) y mutation testing
(caro de correr, para una suite que ya es sólida).

## Consecuencias

- Las unitarias y la integración contra Postgres están implementadas. Los demás tipos
  se implementan en sus propias tareas; el skill `pruebas-vitest` distingue el estado.
- No se versionan scripts de verificación sueltos. Lo que hoy hace un script
  descartable (el recorrido de T23, el volumen de referencia, la medición de RNF-01)
  pasa a ser una prueba de integración, de E2E o de carga.
- El compose suma servicios bajo un *profile* (`backend-e2e`, `frontend-e2e`, `e2e` y el
  de k6). No cambian `npm run up` ni `npm run dev`.
- Hay dos bases más en el Postgres local (`nutria_test`, `nutria_e2e`), que las pruebas
  recrean en cada corrida.
- Las pruebas que llaman al proveedor de IA real (carga de la generación y evaluaciones)
  tienen costo por token y no corren en cada cambio.
- Cuando exista CI (T12), los workflows toman estos mismos comandos.
