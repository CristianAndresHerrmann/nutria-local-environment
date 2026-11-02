# Cierre del Incremento 3 — revisión final

Revisión final del Incremento 3 (T42): criterios de cierre de T25 a T41, DA-12 y las
mejoras que entraron (T26, T29, auditoría técnica), destino de lo que no entró y
despliegue en staging. Trazabilidad: Plan de Proyecto 5.3 (registro de decisiones) y 9
(monitoreo y control); RNF-08 y RNF-13.

- El registro de decisiones del incremento (DA-25 a DA-38, DA-12 resuelta y RNF-14)
  está en [`registro-de-decisiones-incremento-3.md`](registro-de-decisiones-incremento-3.md).
- El fundamento de cada decisión está en [`decisiones/`](decisiones/_indice.md).
- El detalle de cada tarea está en los `CHANGELOG.md` de `nutria-backend`, `nutria-ia` y
  `nutria-frontend`, versión `0.0.3`. En este cierre se cortó también la `0.0.2`
  (Incremento 2), que había quedado en "Unreleased".

**Fecha de cierre:** verificación local el 01/11/2026. El despliegue en staging queda
como último paso, con su checklist en [Despliegue en staging](#despliegue-en-staging).

✅ verificado con código, tests, contra el stack o a mano · ⏳ pendiente, con destino

## Criterios de cierre de las tareas del incremento

| Tarea | Criterio de cierre | Estado | Evidencia |
|---|---|---|---|
| **T31** Rol administrador y cuentas | En staging, el admin crea la cuenta de una nutricionista y ella inicia sesión | ✅ local · ⏳ staging | Recorrido de T42 contra la API local. En staging, en el checklist de abajo |
| | Una cuenta desactivada no inicia sesión; reactivada, sí | ✅ | `auth.service.test.js`, `auth.service.int.test.js`. En el recorrido, el token de la cuenta desactivada deja de servir (401) |
| | Pruebas: 403 cruzados por rol, email repetido → 409, cuenta inactiva, rate limit, el admin no se desactiva a sí mismo | ✅ | `app.test.js` ("roles — 403 cruzados"), `openapi.test.js` (403 documentado en cada operación protegida), `rateLimit.test.js`, `create-admin.test.js`. 409 en `email` en el recorrido |
| | DA-07 modificada y decisión registrada | ✅ | DA-25; DA-07 y DA-04 en "Actualización de decisiones anteriores" |
| **T32** Modelo del plan y la validación | `db:reset` reconstruye la base desde cero | ✅ | `npm run db:reset` el 01/11/2026: las 12 migraciones y el seed sin errores |
| | `schema.prisma` vigente y correspondencia con el Anexo A | ✅ | DA-27, [ajustes del Inc. 3](decisiones/ajustes-al-modelo-del-anexo-a-incremento-3.md) |
| | Pruebas: restricciones por tipo de ítem y aislamiento del repositorio de planes | ✅ | `plan.schema.test.js`, `plan.repository.test.js`, `plan.repository.int.test.js` (FK compuestas y `CHECK` contra Postgres) |
| **T27** Composición por tipo de comida | Almuerzo y cena con origen; desayuno y merienda según la entrevista | ✅ con supuesto | `mealComposition.js`. Desayuno y merienda cargados como supuesto de la GAPA (DA-29), porque la entrevista no se hizo |
| | Pruebas: opción completa, faltante, tipo sin composición | ✅ | `mealComposition.test.js` |
| **T33** Contrato del servicio de IA | Propuesta válida con el simulado; 422 ante un campo fuera de contrato; error controlado ante salida inválida | ✅ | `tests/test_plan_proposals.py`, `tests/test_fake_provider.py` |
| | pytest en la CI; contrato en Swagger | ✅ | 85 tests en verde (abajo). Contrato versionado en `nutria-ia/contracts/openapi.json` y `iaProposal.contract.test.js` en el backend |
| **T34** Motor de reglas v1 | Pruebas que disparan y no disparan por regla | ✅ | `directCondition.test.js`, `itemCategories.test.js`, `validatePlan.test.js` |
| | Skill de reglas activo | ✅ | `.agents/skills/reglas-clinicas-deterministicas/` |
| **T35** Reglas por patología | Cada regla con fuente y origen | ✅ | 37 reglas con origen S y fuente con páginas (DA-36, [`reglas-clinicas.md`](reglas-clinicas.md)) |
| | Pruebas por regla y casos de T41 con los hallazgos esperados | ✅ | `pathologyRule.test.js`, `clinical-rules.test.js`, `clinical-cases.rules.test.js`. En el recorrido, agregar Pan francés a un paciente celíaco da un bloqueo `pathology_rule` |
| **T36** Adaptadores Gemini y OpenAI | Con cada proveedor, un caso de T41 devuelve una propuesta que cumple el contrato | ✅ | 21 de 21 propuestas válidas con cada proveedor en el benchmark del 01/11/2026 ([mediciones](mediciones-de-generacion/README.md)) |
| | CI en verde sin claves | ✅ local | `test_gemini_provider.py` y `test_openai_provider.py` con el cliente simulado, sin red. El estado de la CI en GitHub se revisa antes del merge a `staging` |
| **T37** Generación en el backend | Generación de punta a punta con borrador, hallazgos y traza | ✅ | `planGeneration.service.test.js`, `plan.repository.int.test.js`; en el recorrido, con el proveedor real |
| | El contexto enviado no lleva campos identificatorios | ✅ | `buildContext.test.js`, `allowedCatalog.test.js` |
| | Plan ajeno → 404; falla de la IA → 503 sin escribir | ✅ | `plan.service.test.js`, `planGeneration.service.test.js`; aislamiento en el recorrido |
| **T28** Catálogo base de alimentos | `db:reset` con el catálogo completo; prueba de los datos del seed; origen documentado | ✅ | Seed: 44 grupos, 231 alimentos. `catalogs.test.js`. DA-32 y [`catalogo-de-alimentos.md`](catalogo-de-alimentos.md) (origen V) |
| **T30** Recetas básicas comunes | `db:reset` con las recetas; visibles para los dos nutricionistas; RF-28 las filtra igual | ✅ | Seed: 17 recetas comunes. `recipe.service.test.js`, `recipes.test.js`, `app.int.test.js` (comunes visibles, privadas solo para su dueño). DA-34 |
| **T39** Generación y vista en el frontend | Recorrido desde la ficha hasta la propuesta con hallazgos; reintento ante un error de la IA; pruebas de mapeos | ✅ | `PatientDetail.test.jsx`, `usePlanGeneration.test.jsx`, `patientPlans.test.js`, `src/api/patientPlans.test.js`. Desde las pantallas, en el checklist de staging |
| **T40** Edición y revalidación | Reemplazar un contraindicado levanta el bloqueo; agregarlo lo genera; reemplazar un no reconocido lo resuelve; un cambio en las condiciones se ve al reabrir | ✅ | `app.int.test.js` (T40, contra Postgres), `PlanDetail.test.jsx`, `planForm.test.js`; en el recorrido, con dos pacientes celíacos |
| **T41** Casos ficticios y script de medición | Casos presentes después de `db:reset`; primera medición con el simulado | ✅ | `clinical-cases.test.js`, `measure-generation.test.js`; JSON del simulado y de los dos proveedores en `mediciones-de-generacion/` |
| **DA-12** Proveedor de IA | Resultados versionados y DA-12 registrada | ✅ | DA-12 en el registro; [ficha](decisiones/proveedor-de-ia.md) |
| **T25** Curado del catálogo por el admin | Corregir las categorías de un privado de punta a punta; 400 por ciclo; el plan conserva la copia; 409 al editar y al pasar al común | ✅ | `app.int.test.js` (T25, contra Postgres), `foodAdmin.service.test.js`, `adminCatalog.test.js`, `PromoteFoodForm.test.jsx`. DA-37 |
| **T26** Crear alimento desde la receta (backlog) | Alta desde el buscador de ingredientes, con 409 en el campo | ✅ | `CreateFoodDialog.test.jsx`, `RecipeIngredientsFields.test.jsx` |
| **T29** Unificar errores de formulario (backlog) | Mismo comportamiento en todos los formularios | ✅ | `useFormErrors.test.jsx`. Resuelve la incidencia de pantalla del cierre del Inc. 2 |
| **Auditoría técnica** (AUD-003, 004, 005, 008, 009, 010, 013, 015, 016 y 017) | Hallazgos corregidos con prueba | ✅ | Entradas `AUD-*` de los CHANGELOG `0.0.3`; AUD-016 en [su ficha](decisiones/revocacion-de-tokens-al-restablecer-contrasena.md) (DA-35) |
| **T42** Integración, staging y cierre | Suites en verde, recorrido, aislamiento, staging y documentación | ✅ local · ⏳ staging | Este documento |

## Lo que no entró y su destino

| Ítem | Estado al cierre | Destino |
|---|---|---|
| RAG sobre bibliografía nutricional (DA-13) | No entró (DA-38): se priorizó desplegar para tener el feedback de las nutricionistas | Incremento 4, cards T45 a T51. La línea de base sin RAG ya está medida |
| RF-39: generación de recetas | No entró (DA-38) | Incremento 4. Sin card todavía |
| RNF-14 (generación en 15 s o menos, P95) | Formalizado. La medición preliminar no lo cumple con la configuración vigente (ver abajo) | Medición definitiva en el Incremento 4. Para cumplirlo, adoptar la plantilla v2 (pendiente de revisión profesional, DA-12) |
| Indicador del 95 % de propuestas sin bloqueos | Preliminar: 95,2 % | Medición del Incremento 4 (Tabla 7) |
| Entrevista de validación con la nutricionista | No se hizo | Confirma la matriz de reglas (DA-36, origen S), la composición de desayuno y merienda (DA-29), las categorías V del catálogo (DA-32) y DA-15, DA-16 y DA-22. Sigue con el feedback de staging (T43) |
| T38: Figma de las pantallas del Inc. 3 | No hecho (recortable) | Capturas de las pantallas implementadas para el IA3, como prevé la card |
| RNF-01 con planes en la ficha | No se repitió | Prueba de carga con k6 (DA-24), todavía sin implementar |

## Destino de las cards

Lo informa este documento; el tablero lo actualiza quien gestiona el proyecto.

| Card | Lista al cierre | Destino |
|---|---|---|
| T25, T26, T27, T28, T29, T30, T31, T32, T33, T34, T35, T36, T37, T39, T40, T41, DA-12 y el skill de reglas | Done | Cerradas |
| T42 — Integración, staging y revisión final | Iteration | Se cierra con el checklist de staging marcado |
| T38 — Figma del Inc. 3 | Iteration | Recortada: capturas para el IA3 |
| T43 — Feedback de las nutricionistas | Pending / Blocked | Empieza con este despliegue en staging |
| T44 — Informe de Avance 3 | Iteration | Entrega del 04/11/2026; insumos: este documento y el registro de decisiones |
| T45 a T51 — RAG | Backlog | Incremento 4 |

Sin card todavía, a crear en el Backlog:

- **RF-39: generación de recetas**, para el Incremento 4.
- **Límite de generaciones por nutricionista.** Hoy solo el login tiene límite. En
  staging, con la clave real, un uso abusivo genera costo. Mientras tanto se mitiga con un
  tope de gasto en la clave de OpenAI.
- **Que el smoke test del CD detecte el proveedor simulado.** Hoy `/health/ia` responde
  igual con `fake` que con `openai`. Mientras tanto se verifica con la consulta de la
  sección "Operación" del README del backend.

---

## Estado de las pruebas

Corridas el 01/11/2026 dentro de los contenedores, desde la raíz `nutria/`:

| Repo | Suite | Resultado |
|---|---|---|
| `nutria-backend` | `npm run test:backend` (Vitest: dominio, servicios, repositorios, schemas, serializers, middlewares, spec OpenAPI, contrato con la IA y la app completa sin base) | 72 archivos, 1257 tests en verde |
| `nutria-backend` | `npm run test:backend:integration` (API real y repositorios contra Postgres, base `nutria_test`) | 7 archivos, 58 tests en verde |
| `nutria-ia` | `npm run test:ia` (pytest: contrato, proveedor simulado, adaptadores con el cliente simulado, plantillas, OpenAPI) | 8 archivos, 85 tests en verde |
| `nutria-frontend` | `npm run test:frontend` (Vitest con `jsdom`: cliente HTTP, mapeos, hooks, componentes y páginas) | 32 archivos, 312 tests en verde (después de quitar el aviso de entorno y su test) |

Al cierre del Incremento 2 el backend tenía 671 tests y el frontend 139, y `nutria-ia` no
tenía tests ni había pruebas contra una base real.

## Recorrido y aislamiento contra la base real

Base recreada con `npm run db:reset` (12 migraciones y el seed) y administrador creado
con `npm run db:create-admin`. El recorrido se hizo con un script de cliente HTTP
descartable, que no se versionó, ejecutado dentro del contenedor del backend contra la
API real. Usó el proveedor real (GPT-6 Luna, `low`). Resultado: 41 verificaciones, todas OK.

| Paso | Estado | Qué se verificó |
|---|---|---|
| El admin crea una cuenta | ✅ | Login del admin, alta (201) y email repetido (409 en `email`) |
| La nutricionista inicia sesión | ✅ | Login con la cuenta recién creada; ve el catálogo común (231 alimentos) |
| Paciente de T41 | ✅ | Una cuenta nueva no tiene los pacientes del seed, así que se cargó por la API un paciente con los datos y la condición del caso C1 (celiaquía). Además se repitió todo con el C1 del seed en `nutricionista@nutria.com` |
| Generar | ✅ | 201 con borrador, hallazgos y traza (`openai`/`gpt-6-luna`). Duración: 16,9 s y 11,7 s |
| Ver hallazgos | ✅ | El plan aparece en el listado del paciente y abrirlo lo revalida. Primera propuesta: un ítem no reconocido (bloqueante), una regla moderada de celiaquía y advertencias de composición |
| Editar | ✅ | Agregar Pan francés da un bloqueo `pathology_rule` sobre ese ítem; el texto editado se guarda |
| Revalidar | ✅ | Reemplazar los ítems bloqueados (el no reconocido y el Pan francés) por Arroz blanco deja el plan sin bloqueos de ítem |
| Aislamiento del plan con los dos nutricionistas del seed | ✅ | `nutricionista2@nutria.com` recibe 404 al ver, editar y listar los planes del paciente ajeno, al generar sobre él y al ver el paciente. Tampoco alcanza el plan bajo un paciente propio, ni la cuenta creada por el panel. El listado no muestra pacientes ajenos, y el plan del dueño queda igual después de los intentos |
| Roles y sesión | ✅ | El admin recibe 403 en planes y al generar; una nutricionista recibe 403 en `/admin`. Sin token, 401 en las tres rutas de planes. La cuenta desactivada pierde la sesión en la siguiente request |

La base local queda con los datos del recorrido: la cuenta "Recorrido Incremento Tres"
(desactivada), su paciente y dos borradores. `npm run db:reset` la deja como el seed.

### Verificación manual

Desde las pantallas, con el stack local o en staging:

- [ ] Generación, vista de la propuesta (hallazgos junto al ítem, leyenda de propuesta
      editable), error de la IA con reintento, edición y revalidación.
- [ ] El mismo recorrido en staging, por HTTPS (checklist de abajo).

Esta revisión no recorrió las pantallas: lo que se ve en el navegador lo cubren los
tests del frontend con `jsdom` y esta verificación manual.

## Medición preliminar de RNF-14 (meta de 15 s)

RNF-14 formaliza la meta de la Tabla 7 del IA2: tiempo desde la solicitud hasta la
propuesta validada en el editor, 15 s o menos en P95 (ver el
[registro](registro-de-decisiones-incremento-3.md#requerimiento-no-funcional-nuevo)). Se
midió el 01/11/2026 en local, con los 7 casos de T41 y 3 generaciones por caso, por el
backend ([método y resultados](mediciones-de-generacion/README.md)).

| Configuración (GPT-6 Luna) | Total P95 | ¿Cumple? |
|---|---|---|
| Razonamiento `low`, plantilla v1 (la vigente) | 16,4 a 20,1 s en cuatro corridas | ✗ |
| Razonamiento `low`, plantilla v2 | 13,9 y 11,1 s en dos corridas | ✓, pendiente de revisión profesional |
| Razonamiento `none`, plantilla v1 | 10,5 s | ✓, descartada por calidad (DA-12) |

El overhead del backend (contexto, validación y escritura) es de unos 0,1 s. El tiempo
es del proveedor y depende de los tokens de salida. En el recorrido de T42, con la
configuración vigente, una generación tardó 16,9 s y otra 11,7 s. Es una medición
preliminar: local, sin concurrencia y con N=3. La definitiva es del Incremento 4.

## Incidencias y destino

| Incidencia | Destino |
|---|---|
| El `nutria-ia/.env` de la VM tiene las variables del Incremento 2. Sin `AI_PROVIDER`, el servicio arranca con el proveedor simulado y el smoke test del CD pasa igual. El README decía que ese `.env` no se usaba en staging | Resuelta en la documentación: README del backend, sección "Staging". Se corrige en la VM antes del despliegue (checklist) |
| El despliegue migra pero no corre el seed, y sin el seed staging no tiene el catálogo, las reglas, las recetas comunes ni los casos de T41 | Paso del checklist; agregado a "Operación" en el README |
| El recorrido de la card pide un paciente de T41 en la cuenta creada por el panel, que no lo tiene | Se cargó un paciente equivalente por la API, y el recorrido se repitió con el paciente del seed |
| Los CHANGELOG tenían el Incremento 2 sin cortar, y T31, que es del Incremento 3, ya había llegado a staging | Cortadas `0.0.2` (lo que estaba en staging, sin T31) y `0.0.3` en los tres repos |
| La generación no tiene límite por nutricionista, y en staging corre con la clave real | Card a crear (arriba). Mitigación: tope de gasto en la clave |
| En staging, el aviso fijo "Entorno de prueba — solo datos ficticios" molestaba, y su reserva de altura dejaba un salto al final de cada pantalla | Resuelta: se quitó el aviso (`EnvironmentBanner` y `VITE_ENV_LABEL`). RNF-08 queda cubierto por el recordatorio al dar cada cuenta |
| El estado de la CI en GitHub no se pudo consultar desde el entorno de esta revisión | Se revisa antes de abrir las PRs a `staging` (checklist) |

## Limitaciones

- **Sin E2E en navegador ni prueba de carga.** Siguen definidas en la
  [estrategia de pruebas](decisiones/estrategia-de-pruebas.md). Las pantallas se validaron
  a mano y la API, contra Postgres.
- **Reglas y catálogo sin revisión profesional** (origen S y V), hasta la entrevista de
  validación.
- **RNF-14 medido solo en local**, con N=3 y sin concurrencia.

## Despliegue en staging

Lo ejecuta quien inicia la sesión, en este orden. Comandos completos en el README de
`nutria-backend`, sección "Staging" (`dcs` es el alias definido ahí). Solo datos
ficticios (RNF-08).

**Antes del merge a `staging`**

- [ ] CI en verde en `development` en los tres repos.
- [ ] VM *Running* en la consola de OCI. Por SSH: `dcs ps`, espacio en disco
      (`df -h /`, `docker system df`).
- [ ] Backup de la base (`pg_dump`, ver "Operación") y foto de las cuentas
      (`select email, role, active, created_at from nutritionist`).
- [ ] `nutria-ia/.env` con el proveedor real y la clave, y tope de gasto en la clave.
- [ ] La clave responde desde la VM (`curl` a `https://api.openai.com/v1/models` → 200).

**Merge y despliegue**

- [ ] PR `development → staging` en `nutria-ia`; CD en verde.
- [ ] PR `development → staging` en `nutria-backend`; CD en verde (levanta todo el stack y migra).
- [ ] PR `development → staging` en `nutria-frontend`; CD en verde.

**Después del despliegue**

- [ ] Las 7 migraciones nuevas en `_prisma_migrations`.
- [ ] Seed recargado (`dcs run --rm backend npm run seed`).
- [ ] Las cuentas creadas desde el panel siguen intactas (misma consulta que antes del despliegue).
- [ ] Smoke test por HTTPS: frontend, `/health` y `/health/ia` (los corre el CD).
- [ ] Recorrido por HTTPS desde las pantallas: el admin crea una cuenta, la nutricionista
      inicia sesión, carga un paciente ficticio, genera, ve los hallazgos, edita y revalida.
- [ ] La generación quedó con `provider = openai` en `ai_generation`.
- [ ] Recordatorio a las nutricionistas: solo pacientes ficticios. La interfaz ya no muestra el aviso de entorno de prueba (se quitó al cierre), así que el recordatorio es el único resguardo.

Resultado del despliegue (completar): fecha, commits desplegados e incidencias.
