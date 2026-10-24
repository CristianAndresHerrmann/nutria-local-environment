# Cierre del Incremento 2 — verificación conjunta (T23)

Verificación conjunta de T14 a T22 antes de la revisión final: pruebas
obligatorias, recorrido desde base vacía, documentación de la API y una medición
preliminar de RNF-01. Trazabilidad: Plan de Proyecto 5.6 (validación); RNF-01,
RNF-05, RNF-08 y RNF-13. El detalle de cada tarea está en los `CHANGELOG.md` de
`nutria-backend` y `nutria-frontend`, en "Unreleased". Las decisiones técnicas
están en [`decisiones/`](decisiones/_indice.md).

✅ verificado con código, tests o contra el stack · 🔸 requiere verificación manual

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
| Errores junto al campo, en cada formulario | ✅ / 🔸 | Revisión de código: paciente, condición, consulta (con sus mediciones y hábitos), receta (con ingredientes y tipos de comida), recomendación y alimento muestran cada error de `details.fields` junto a su control. Cada campo que el backend puede señalar tiene dónde mostrarse. La pestaña de mediciones es de solo lectura. Falta verlo en el navegador |
| Mismo recorrido desde las pantallas | 🔸 | Pendiente, con el stack levantado y el usuario del seed |

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

Con `npm run db:seed:volume` (100 pacientes, de 1 a 20 consultas cada uno, 1050
consultas en total) y `npm run perf:patient-detail`. La ficha completa son las
cuatro requests de la ficha, en paralelo: paciente, condiciones, consultas y
mediciones.

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
  pantallas se verifican a mano (ver los 🔸).
- **RNF-01 medido solo en local**, como medición preliminar (ver arriba).
- Ya figuran en el CHANGELOG del backend: la composición esperada por tipo de
  comida (RF-26), que se recortó de T15, y la copia del alimento en el ítem del
  plan.
- La base local queda con los datos del recorrido (pacientes "Recorrido
  Incremento Dos") y la cuenta `volumen@nutria.com`. `npm run db:reset` la deja
  como el seed.

### Verificación manual pendiente

Con el stack levantado según el [README](../README.md#primer-arranque) y el
usuario del seed:

- [ ] Recorrido desde las pantallas: alta de paciente → datos clínicos y
      contextuales → condiciones → consulta con mediciones y hábitos → receta →
      recomendación.
- [ ] En cada formulario, provocar un error de validación y ver el mensaje junto
      al campo: paciente, condición, consulta (con una medición inválida y un
      hábito vacío), receta (sin tipos y sin ingredientes), recomendación y
      alimento.
