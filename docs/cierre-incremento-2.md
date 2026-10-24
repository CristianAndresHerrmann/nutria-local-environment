# Cierre del Incremento 2 — revisión final

Revisión final del Incremento 2 (T24): criterios de cierre de T14 a T23, destino de
lo que no entró y pendientes que condicionan el Incremento 3. Trazabilidad: Plan de
Proyecto 5.3 (registro de decisiones) y 9 (monitoreo y control). Desde
[Estado de las pruebas](#estado-de-las-pruebas) sigue la verificación conjunta de
T23.

- El registro de decisiones del incremento (DA-15 a DA-24), la Tabla 3 actualizada y
  el modelo lógico, listos para el Informe de Avance 3, están en
  [`registro-de-decisiones-incremento-2.md`](registro-de-decisiones-incremento-2.md).
- El fundamento de cada decisión está en [`decisiones/`](decisiones/_indice.md).
- El detalle de cada tarea está en los `CHANGELOG.md` de `nutria-backend` y
  `nutria-frontend`, en "Unreleased".

**Fecha de cierre:** prevista para el 18/10/2026 en el Informe de Avances 2,
reprogramada; cierre real el 25/10/2026.

✅ verificado con código, tests, contra el stack o a mano desde las pantallas

## Criterios de cierre de T14 a T23

| Tarea | Criterio de cierre | Estado | Evidencia |
|---|---|---|---|
| **T14** Modelo de datos | Base reproducible desde cero con migrar + seed; migración con nombre descriptivo, sin `db push` | ✅ | `npm run db:reset` en T23: las 4 migraciones desde cero y el seed sin errores. Las 3 migraciones del incremento tienen nombre descriptivo |
| | `schema.prisma` actualizado como modelo lógico vigente | ✅ | `nutria-backend/prisma/schema.prisma`; correspondencia con el Anexo A en [la decisión de ajustes](decisiones/ajustes-al-modelo-del-anexo-a-incremento-2.md) |
| **T15** Alimentos, grupos y patologías | Alta de un alimento con dos categorías dentro de un subgrupo, de punta a punta | ✅ | `food.service.test.js` (dos categorías en "Pescados") y `app.test.js` (ruta completa, sin base). Validado a mano desde la pantalla |
| | Un alimento privado no es visible para otro nutricionista | ✅ | Tests de servicio, repositorio y rutas. Seed: "Kéfir", privado de `nutricionista@nutria.com` |
| | Un alimento en uso no puede eliminarse físicamente | ✅ | La API no tiene `DELETE` (`app.test.js` da 404) y las FK de condiciones e ingredientes son `RESTRICT` |
| | Pruebas: categorías válidas, jerarquía sin ciclos, duplicado → `CONFLICT`, aislamiento | ✅ | `food.service.test.js`, `foodGroupTree.test.js`, `food.repository.test.js`, `app.test.js` |
| **T16** Datos clínicos y contextuales | Un paciente se guarda con algunos campos clínicos y se completa después | ✅ | Tests de guardado parcial; alta con datos clínicos en el recorrido de T23 |
| | La ficha muestra "Sin dato" en los campos vacíos, sin errores | ✅ | `PatientDetail.jsx` muestra "Sin dato" ante un valor vacío. Validado a mano en la ficha |
| | Pruebas: aislamiento y error por campo | ✅ | `patient.service.test.js`, `patient.schema.test.js`, `app.test.js` |
| **T17** Selector de fecha | Fecha de nacimiento, de consulta y de medición usan el mismo componente | ✅ | `DateInput` en `PatientForm` y `ConsultationForm`. Desde la interfaz, una medición se carga dentro de la consulta y toma su fecha (la pestaña de mediciones es de solo lectura) |
| | 30/02 enviado a la API → 400 con detalle por campo | ✅ | `common.schema.test.js`, `app.test.js` y el recorrido de T23 |
| **T18** Condiciones | Una alergia a un alimento, un rechazo a un grupo y una patología se registran y se ven en la ficha | ✅ | Recorrido de T23 contra la API, y validado a mano en la ficha |
| | Pruebas: combinaciones inválidas, duplicado → `CONFLICT`, aislamiento | ✅ | `patientCondition.schema.test.js`, `patientCondition.service.test.js`, `patientCondition.repository.test.js`, `app.test.js` |
| **T19** Consultas y hábitos pautados | Una primera consulta y un seguimiento con hábitos pautados se ven en el historial | ✅ | Recorrido de T23 y `consultation.service.test.js` |
| | Una consulta sin mediciones se guarda sin errores (RNF-09) | ✅ | `consultation.service.test.js`, `consultation.schema.test.js` |
| | Pruebas: aislamiento, tipo y fecha | ✅ | `consultation.service.test.js`, `consultation.repository.test.js`, `app.test.js` |
| **T20** Mediciones y laboratorio | Peso, talla y un valor de laboratorio en una consulta, con el IMC en el histórico | ✅ | Recorrido de T23: IMC 24,2 y 23,7 con la talla de la primera consulta. Validado a mano en la pestaña de mediciones |
| | Pruebas del IMC: con peso y talla, sin talla, con talla anterior | ✅ | `src/utils/bmi.test.js` (9 casos) |
| | Pruebas de aislamiento y de `detail` obligatorio | ✅ | `measurement.service.test.js`, `measurement.schema.test.js`, `app.test.js` |
| **T21** Recetas | Receta con tres ingredientes y dos tipos de comida, creada y editada | ✅ | `recipe.service.test.js` |
| | Receta de otro nutricionista → 404 | ✅ | `recipe.service.test.js`, `app.test.js` y el recorrido de T23 |
| | RF-28 (entró): alimento directo, grupo, subgrupo, paciente sin condiciones | ✅ | `recipeConditionFilter.test.js` y el recorrido de T23 |
| **T22** Recomendaciones | Alta, edición y baja lógica; filtro por patología | ✅ | `recommendation.service.test.js`, `app.test.js`; alta propia y 403 sobre una común en el recorrido de T23 |
| **T23** Integración | Suite en verde en backend y frontend | ✅ | 671 y 139 tests (24/10/2026), ver abajo |
| | Recorrido completo sin errores; incidencias resueltas o registradas con destino | ✅ | Recorrido contra la API completo y sin errores. El mismo recorrido desde las pantallas se validó a mano antes de integrar cada tarea a `development` (ver [verificación manual](#verificación-manual)) |

## Lo que no entró y su destino

La card de T24 nombraba cuatro candidatos a recorte. Solo RF-26 quedó afuera:

| Ítem | Estado al cierre | Destino |
|---|---|---|
| RF-26: composición esperada por tipo de comida | No entró: se recortó de T15. Almuerzo y cena se conocen (verdura, proteína e hidrato); desayuno y merienda dependen de la entrevista de validación | Incremento 3, como configuración que consume el motor de reglas. No tiene card propia todavía |
| RF-28: filtro de recetas por condiciones | Entró en T21 (DA-23) | — |
| T22: recomendaciones clínicas | Entró, con recomendaciones privadas por nutricionista (DA-16) | — |
| RNF-01: medición preliminar de la ficha | Hecha en T23: P95 de 22 ms en local, contra un umbral de 5 s | La medición que cuenta es la prueba de carga con k6 (DA-24), al cerrar cada incremento y en staging. Se repite cuando la ficha incluya planes |

## Destino de las cards

| Card | Lista | Destino |
|---|---|---|
| T14 a T22 | Done | Cerradas |
| T23 — Probar integración completa | Testing | Se cierra: recorrido desde las pantallas validado |
| T24 — Documentar decisiones y revisión final | Working | Se cierra con este documento y el registro de decisiones |
| T25 — Rol administrador del catálogo | Iteration | No formaba parte del Incremento 2: la abrió T15. Falta asignarle incremento. No condiciona el Incremento 3, pero sí la corrección de alimentos en uso, que depende de la copia del alimento en el ítem del plan |
| T26 — Crear alimento desde el formulario de receta | Backlog | Mejora de usabilidad, prioridad baja, sin incremento asignado |
| Investigar proveedores de IA (DA-12) | Pending / Blocked | Antes del inicio del Incremento 3 |
| T12 — CI/CD | Backlog | Sin cambios: la CI se puede hacer en cualquier momento; el CD, con la card de staging |
| Desplegar staging bajo demanda | Backlog | Cuando exista el servicio de generación de planes (DA-14) |
| Skill de reglas clínicas determinísticas | Backlog | Incremento 3, después de la entrevista de validación |
| Definir flujo de ramas y plantilla de PR | Backlog | Sin cambios: siguen faltando `staging` y la plantilla de PR en los tres repos |

Sin card todavía, y a crear en el Backlog:

- **Composición esperada por tipo de comida (RF-26)**, para el Incremento 3.
- **Catálogo base de alimentos y recetas básicas en el seed**, para el Incremento 3,
  antes de evaluar la generación (DA-21). Las recetas básicas comunes a todos los
  nutricionistas cambian DA-15, donde toda receta es de un nutricionista: hay que
  decidir cómo se modelan.
- **Errores repetidos arriba del botón**: en los formularios de paciente, receta y
  alimento, un error por campo se repite arriba del botón, y en los de condición,
  consulta y recomendación no (ver incidencias de T23). Prioridad baja; falta decidir
  qué comportamiento se unifica.

## Pendientes que condicionan el Incremento 3

- **Entrevista de validación con la nutricionista.** Define la matriz de reglas
  clínicas (RF-47) y la composición de desayuno y merienda (RF-26). Sin ella no se
  implementa el motor de reglas. Además confirma los supuestos de DA-15, DA-16 y DA-22.
- **Selección del proveedor de IA (DA-12).** Los candidatos principales son Gemini
  3.8 Flash y GPT-6 Luna; falta comparar costos y ejecutar el benchmark propio. Hay
  que registrar la decisión antes de empezar el servicio de generación.
- **Volumen del catálogo de alimentos (DA-21).** La propuesta de la IA se resuelve
  contra el catálogo: con 31 alimentos comunes, muchos ítems no se reconocerían. El
  catálogo definitivo (materia prima, de 150 a 250 alimentos) lo arma el desarrollador
  a partir de la bibliografía y de las conversaciones con nutricionistas. Se cruza con
  el punto abierto del IA2 sobre los ítems no reconocidos.
- **Copia del alimento y de la recomendación en el plan (DA-16).** Es un requisito de
  diseño de OPCION_ITEM y del plan, no una tarea aparte.

---

## Estado de las pruebas

Corridas el 24/10/2026 dentro de los contenedores (`npm run test:backend`,
`npm run test:frontend`):

| Repo | Suite | Resultado |
|---|---|---|
| `nutria-backend` | Vitest: servicios con repositorios fake, repositorios con Prisma fake, schemas, serializers, utils, middlewares, spec OpenAPI, y la app completa sin base (`src/app.test.js`) | 42 archivos, 671 tests en verde |
| `nutria-frontend` | Vitest: cliente HTTP, mapeos de cada formulario, fechas, árbol de grupos | 11 archivos, 139 tests en verde |
| `nutria-ia` | — | Sin tests (ver limitaciones) |

Al empezar T23, la suite del backend tenía 627 tests. Los 44 nuevos cubren lo
que se corrigió en esta tarea (ver incidencias).

## Pruebas unitarias obligatorias

| Caso | Estado | Evidencia |
|---|---|---|
| Aislamiento por nutricionista: condiciones (recurso ajeno → 404) | ✅ | `patientCondition.service.test.js`: listar, crear, editar y borrar sobre un paciente ajeno, y una condición de otro paciente. `patientCondition.repository.test.js`: el nutricionista va en el `where` de cada consulta, y el repositorio corta si no viene. `app.test.js`: el nutricionista del token llega al servicio, y sin token responde 401 |
| Aislamiento: consultas | ✅ | `consultation.service.test.js` (listar, ver, crear y editar), `consultation.repository.test.js`, `app.test.js` |
| Aislamiento: mediciones | ✅ | `measurement.service.test.js` (listar, crear, editar y borrar; la consulta de otro paciente da 400), `measurement.repository.test.js`, `app.test.js` |
| Aislamiento: recetas | ✅ | `recipe.service.test.js` (ver y editar una ajena; el listado trae solo las propias; el filtro RF-28 con un paciente ajeno), `recipe.repository.test.js`, `app.test.js` |
| Aislamiento: recomendaciones (T22, fuera de la lista de la card) | ✅ | `recommendation.service.test.js`, `recommendation.repository.test.js`, `app.test.js` |
| Validaciones de PACIENTE_CONDICION: combinaciones por tipo | ✅ | `patientCondition.schema.test.js`, más 4 combinaciones inválidas de punta a punta en `app.test.js` |
| Validaciones de PACIENTE_CONDICION: duplicados | ✅ | `patientCondition.service.test.js`: 409 al crear y al editar, la propia condición no cuenta, y otro tipo o el mismo tipo en otro paciente no son duplicados |
| Cálculo del IMC | ✅ | `src/utils/bmi.test.js` (9 casos) y `measurement.service.test.js` |
| Filtro de recetas por condiciones (RF-28, entró en T21) | ✅ | `src/utils/recipeConditionFilter.test.js` y `recipe.service.test.js` |

No faltaba ninguna. Los casos de aislamiento usan fakes, así que también se
verificaron contra Postgres en el recorrido.

## Recorrido desde base vacía

Base recreada con `npm run db:reset`: las 4 migraciones se aplican desde cero y
el seed corre sin errores. El recorrido se hizo con un script de cliente HTTP
contra `api.local.nutria.com`, con los dos nutricionistas del seed: 65
verificaciones, todas OK. Después de los arreglos de T23 se reconstruyó el stack y
el recorrido se repitió, con el mismo resultado.

| Paso | Estado | Qué se verificó |
|---|---|---|
| Migrar + seed | ✅ | `db:reset` sin errores. Catálogo: 15 grupos, 31 alimentos, 3 patologías y 3 recomendaciones comunes |
| Alta de paciente con datos clínicos y contextuales | ✅ | 201. La ficha avisa que faltan datos clínicos (RF-45). Una fecha inexistente da 400 en `birthDate` |
| Condiciones | ✅ | Patología, alergia, intolerancia al mismo alimento y rechazo a un grupo. La alergia repetida da 409. Alimento y grupo a la vez, o una patología sin `pathologyId`, dan 400 en su campo. Con patología y alergia cargadas, el aviso se apaga |
| Consulta con mediciones y hábitos | ✅ | Una primera consulta con hábito y peso, talla y glucemia, y un seguimiento. Errores en su campo: pliegue sin detalle, fecha futura, unidad que no corresponde al tipo, valor 0. El historial va de la más reciente a la más antigua. El IMC da 24,2 y 23,7, con la talla de la primera consulta |
| Receta | ✅ | Una receta apta y una con merluza. Al paciente que rechaza "Carnes, pescados y huevos", el filtro RF-28 le deja afuera la de merluza (grupo Pescados, subgrupo del rechazado). Una receta sin tipos de comida da 400 en `mealTypes` |
| Recomendación | ✅ | Recomendaciones de la patología del paciente, alta de una propia, y 403 al editar una común |
| Aislamiento contra la base real | ✅ | El segundo nutricionista intentó 19 operaciones sobre datos del primero (paciente, condiciones, consultas, mediciones, recetas, filtro RF-28 y recomendación): todas dan 404. Los listados no muestran nada ajeno, y después de los intentos los datos del primero siguen iguales |
| Sin token | ✅ | 401 en los cinco listados nuevos |
| Errores junto al campo, en cada formulario | ✅ | Revisión de código: paciente, condición, consulta (con sus mediciones y hábitos), receta (con ingredientes y tipos de comida), recomendación y alimento muestran cada error de `details.fields` junto a su control. Cada campo que el backend puede señalar tiene dónde mostrarse. La pestaña de mediciones es de solo lectura. Validado a mano en el navegador |
| Mismo recorrido desde las pantallas | ✅ | Validado a mano antes de integrar cada tarea a `development` |

## Documentación de la API

✅ Swagger documenta las 34 operaciones de la API. `src/docs/openapi.test.js`
verifica que estén todas, que ninguna sobre y que ningún `$ref` quede colgado.
Además verifica:

- El token y el 401 en las rutas protegidas.
- El 201 de cada alta.
- El 409 de alimentos y condiciones, y el 403 de las recomendaciones comunes.
- Que no se documente el borrado de consultas, recetas y recomendaciones.
- Desde T23, el 400 en todas las operaciones (ver incidencias).

## Medición preliminar de RNF-01 (recortable, hecha)

Medida con un script descartable, que no se versionó: la medición que queda en
el proyecto será el test de carga de la
[estrategia de pruebas](decisiones/estrategia-de-pruebas.md). El volumen fue el
de referencia del RNF-01: 100 pacientes ficticios con 1 a 20 consultas cada uno,
1050 consultas en total, en una cuenta aparte (`volumen@nutria.com`). La ficha
completa son las cuatro requests de la ficha, en paralelo: paciente,
condiciones, consultas y mediciones. Primero una vuelta de calentamiento y
después 5 rondas sobre los 100 pacientes, con un usuario a la vez.

| Medición (24/10/2026, local) | n | P50 | P95 | Máx. |
|---|---|---|---|---|
| Ficha completa | 500 | 14 ms | 22 ms | 33 ms |
| Ficha con 20 consultas | 25 | 15 ms | 24 ms | 24 ms |
| Listado de pacientes | 5 | 10 ms | 11 ms | 11 ms |

La ficha más pesada ocupa 25 KB. El umbral es un P95 menor a 5 s, así que hay
margen de sobra. Es una medición preliminar:

- Corre en la máquina de desarrollo, por la red interna de Docker. No incluye el
  render del navegador ni la latencia del despliegue.
- Desde el host, a través del proxy, da valores parecidos (P95 de 19 ms).

Se repite en el despliegue y cuando la ficha sume planes.

## Incidencias y destino

| Incidencia | Destino |
|---|---|
| Los errores de validación que el schema no personaliza respondían el mensaje por defecto de Zod, en inglés. Por ejemplo, un parámetro de query desconocido o un `nutritionistId` en el body daban `Unrecognized key: "page"`, y un obligatorio ausente, `Invalid input: expected string, received undefined` | Resuelta en T23: mensajes en castellano configurados en `common.schema.js`, con tests |
| 15 operaciones GET y DELETE no documentaban el 400 que devuelven ante un parámetro de query desconocido. Además, la descripción de `ValidationError` decía que un parámetro de ruta inválido da 400, cuando da 404 | Resuelta en T23, con un test que exige el 400 en todas las operaciones |
| Inconsistencia de pantalla: en los formularios de paciente, receta y alimento, un error por campo también se repite arriba del botón. En los de condición, consulta y recomendación, no | Registrada, sin cambio en T23: no rompe ningún criterio. Queda para decidir cuál de los dos comportamientos se unifica |

## Limitaciones y trabajo que pasa al Incremento 3

- **Sin integración continua.** Ningún repo tiene workflows. Sigue en la card de
  Trello "T12 — CI/CD". Las suites se corren a mano, dentro de Docker.
- **`nutria-ia` sigue siendo un esqueleto**: solo `/health`, sin tests ni
  `pytest`.
- **Sin pruebas del frontend en el navegador ni de integración contra una base
  real.** El recorrido de este documento cubre la API contra Postgres. Las
  pantallas se verificaron a mano. La integración, el E2E y la carga
  ya están definidos en la [estrategia de pruebas](decisiones/estrategia-de-pruebas.md)
  y se implementan en tareas propias.
- **RNF-01 medido solo en local**, como medición preliminar y sin concurrencia
  (ver arriba).
- Ya figuran en el CHANGELOG del backend: la composición esperada por tipo de
  comida (RF-26), que se recortó de T15, y la copia del alimento en el ítem del
  plan.
- La base local queda con los datos del recorrido (pacientes "Recorrido
  Incremento Dos") y la cuenta `volumen@nutria.com`. `npm run db:reset` la deja
  como el seed.

### Verificación manual

Con el stack levantado según el [README](../README.md#primer-arranque) y el
usuario del seed, antes de integrar cada tarea a `development`:

- [x] Recorrido desde las pantallas: alta de paciente → datos clínicos y
      contextuales → condiciones → consulta con mediciones y hábitos → receta →
      recomendación.
- [x] En cada formulario, provocar un error de validación y ver el mensaje junto
      al campo: paciente, condición, consulta (con una medición inválida y un
      hábito vacío), receta (sin tipos y sin ingredientes), recomendación y
      alimento.
