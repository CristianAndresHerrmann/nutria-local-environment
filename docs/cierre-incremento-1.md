# Cierre del Incremento 1 — revisión final

Revisión de los criterios de cierre de las tareas T1 a T8, de las tareas de soporte
del incremento, y de las limitaciones y trabajos que pasan al Incremento 2. Las
decisiones técnicas del incremento están en [`decisiones/`](decisiones/_indice.md).

## Estado de las pruebas

| Repo | Suite | Resultado |
|---|---|---|
| `nutria-backend` | Vitest, unitarias (servicios, repositorios, schemas, middlewares) más integración liviana de la app sin base | 14 archivos, 119 tests en verde |
| `nutria-frontend` | Vitest (cliente HTTP, mapeo del formulario de paciente) | 2 archivos, 15 tests en verde |
| `nutria-ia` | — | Sin tests (ver limitaciones) |

Las suites corren dentro de los contenedores con `npm run test:backend` y
`npm run test:frontend` desde la carpeta raíz.

## Criterios de cierre de T1 a T8

✅ verificado con código o tests · 🔸 requiere verificación manual con el stack levantado

| Tarea | Criterio de cierre | Estado | Evidencia |
|---|---|---|---|
| **T1** Crear repositorios y estructura | Los tres repos clonables, con estructura acordada | ✅ | `nutria-backend`, `nutria-frontend`, `nutria-ia` |
| | El README del backend documenta la estructura de carpetas raíz | ✅ | `nutria-backend/README.md`, sección "Estructura de carpetas raíz" |
| **T2** Inicializar proyectos | Cada componente corre localmente con un solo comando | ✅ | `npm run dev` (backend, frontend), `python run.py` (IA) |
| | `/health` de backend e IA responden 200 | ✅ | Backend: `src/app.test.js`. Ambos verificados con el stack levantado |
| **T3** Infraestructura local | `docker compose up` levanta los servicios y `/health` de backend e IA responden | ✅ | Seis servicios arriba; `/health`, `/health/ia` (backend → IA), frontend, Swagger y pgAdmin responden por sus dominios locales |
| | Ningún secreto versionado | ✅ | Solo se versionan los `.env.example`. Las credenciales del seed y de pgAdmin son ficticias y de uso local |
| **T4** Modelo lógico y persistencia | Base reproducible desde cero con migrar + seed | 🔸 | `prisma/migrations/`, `scripts/seed.js`, `npm run db:reset` |
| | Prisma Studio muestra los datos del seed | 🔸 | `npm run db:studio` |
| **T5** Autenticación y control de acceso | Sin token → 401 | ✅ | `src/middlewares/auth.test.js`, `src/app.test.js` |
| | Token válido sobre paciente de otro nutricionista → 404 | ✅ | `src/services/patient.service.test.js`. Verificado también contra la API con los dos nutricionistas del seed |
| | Credenciales inválidas: error uniforme sin revelar si el email existe | ✅ | `src/services/auth.service.test.js` |
| **T6** Gestión básica de pacientes | Flujo alta → listado → ficha → edición contra la API real | ✅ / 🔸 | API verificada con el stack levantado: login, listado, alta parcial, ficha y edición (con registro de `dataCompletedAt`). El recorrido por las pantallas queda para la verificación manual |
| | Un paciente menor no se puede guardar sin responsable | ✅ | `src/schemas/patient.schema.test.js`, `src/app.test.js` |
| | El listado solo muestra pacientes del nutricionista autenticado | ✅ | `src/services/patient.service.test.js`, `src/repositories/patient.repository.test.js` |
| **T7** Integración y manejo de errores | Ningún fallo termina en respuesta vacía, caída del proceso o stack expuesto | ✅ | `src/app.test.js`, `src/middlewares/errorHandler.test.js` |
| | Un error de validación del backend se muestra en el campo correspondiente | ✅ | Cliente HTTP del frontend (`src/api/client.test.js`) y formularios con error por campo |
| **T8** Pruebas unitarias | Suite verde, ejecutable con un comando | ✅ | `npm run test:backend` |
| | Caso de aislamiento entre nutricionistas cubierto y documentado | ✅ | Tests de servicio, repositorio y de cada ruta de pacientes (`src/app.test.js`); [decisión de autenticación y aislamiento](decisiones/autenticacion-jwt-y-aislamiento-por-nutricionista.md) |

### Verificación manual pendiente

Con el stack levantado según el [README](../README.md#primer-arranque):

- [x] `npm run up` levanta todos los servicios y responden `http://api.local.nutria.com/health` e `http://ia.local.nutria.com/health` (T2, T3).
- [ ] `npm run db:reset` recrea la base desde cero y `npm run db:studio` muestra los datos del seed (T4).
- [x] Contra la API, con el usuario del seed: alta parcial → listado → ficha → edición; menor sin responsable → 400; sin token → 401; id inexistente → 404; credenciales inválidas → 401 uniforme (T5, T6).
- [x] Aislamiento con los dos nutricionistas del seed: cada listado muestra solo sus pacientes; leer o editar un paciente del otro responde 404 y no aplica el cambio; un `nutritionistId` en el body se rechaza con 400 (T5, T6, T8).
- [ ] El mismo recorrido de T6 desde las pantallas del frontend.

## Tareas de soporte del incremento

| Tarea | Estado |
|---|---|
| Documentación de la API con Swagger | Cerrada |
| T11 — Contexto y skills para asistentes de IA | Cerrada |
| T13 — Tailwind CSS en el frontend | Cerrada |
| Página para recursos no encontrados (frontend) | Cerrada |
| T10 — Flujo de ramas y plantilla de PR | Flujo en uso; faltan `staging` y la plantilla de PR (ver limitaciones) |
| T12 — CI con GitHub Actions | Pospuesta al Incremento 2 |

## Limitaciones y trabajo que pasa al Incremento 2

- **Sin integración continua (T12).** Ningún repo tiene workflows de CI. Las suites
  se corren a mano, dentro de Docker. Se retoma en el Incremento 2.
- **`nutria-ia` es un esqueleto.** Expone solo `/health`, no tiene tests ni `pytest`
  en sus dependencias (`npm run test:ia` falla por eso) y no hay integración con el
  proveedor de IA generativa. Se desarrolla cuando empiece la funcionalidad de IA.
- **pgvector disponible pero sin activar.** La imagen lo trae; la extensión y la
  columna de embeddings llegan con RAG.
- **Modelo de datos acotado.** Solo nutricionista y paciente con los campos de
  RF-02, RF-03, RF-15 y RF-17. Las columnas clínicas y contextuales (RF-04 a RF-14)
  entran por migración en el Incremento 2.
- **Alta de nutricionistas solo por seed**, sin recuperación de contraseña (ver
  [decisión](decisiones/alta-de-nutricionistas-por-seed.md)).
- **Pruebas unitarias, no end-to-end.** No hay tests contra una base real ni del
  frontend en el navegador. Los flujos completos se verifican a mano (ver arriba).
- **Flujo de ramas incompleto** (ver [decisión](decisiones/flujo-de-ramas.md)): no
  existe `staging` en ningún repo, no hay plantilla de PR versionada, los PR se
  integraron con merge commit en vez de squash, y los cambios de soporte se
  commitearon directo en `development`.
- **Versiones sin publicar.** Los `CHANGELOG.md` de backend y frontend siguen en
  "Unreleased"; la entrada de versión se crea al pasar el incremento a `master`.
