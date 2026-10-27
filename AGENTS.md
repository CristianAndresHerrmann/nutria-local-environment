# NutrIA — contexto para asistentes de IA (Claude Code / Codex)

Este archivo se carga en cada sesión, desde la carpeta raíz `nutria/` (que contiene
los tres repos hijos: `nutria-backend/`, `nutria-ia/`, `nutria-frontend/`, más este
repo chico de tooling). Los procedimientos paso a paso viven en
`.agents/skills/` (symlinkeado como `.claude/skills/` para Claude Code); acá solo
van reglas siempre vigentes.

**Alcance de los asistentes de IA en este proyecto:** exclusivamente desarrollo,
control de versiones, código y pruebas. No gestionan Trello ni la planificación del
proyecto — la tarea, su alcance y su criterio de cierre los define quien inicia la
sesión. El identificador `T<N>` (de Trello) se usa solo como referencia en ramas y
commits.

**Regla general:** todo comando `git`/`gh` corre dentro del repo afectado
(`nutria-backend/`, `nutria-ia/` o `nutria-frontend/`), nunca en la raíz `nutria/`.

## 1. Flujo de ramas y commits (ver skill `flujo-git-nutria` para el detalle)

- Ramas permanentes: `master` (producción) y `development` (integración; todas las
  PRs apuntan acá). `staging` existe desde el inicio y recibe fast-forward (o PR)
  desde `development` cuando una iteración está completa.
- Ramas de tarea desde `development`: `T<N>-TituloTarea`. Una tarea = un PR, squash
  hacia `development`.
- Commits: `tipo(T<N>): descripción`, tipo ∈ {`feat`, `fix`, `chore`, `docs`}.
  Ejemplo: `feat(T1): alguna funcionalidad genial`.
- Nunca push directo a `master`, `development` o `staging`: siempre PR.
- **El historial lo maneja quien inicia la sesión.** El asistente no hace `add`,
  `commit`, `push`, `merge`, `checkout -b` ni abre PRs por iniciativa propia, ni
  propone dividir el trabajo en commits ni sugiere cuándo commitear. Deja los
  cambios sin commitear. Solo ejecuta una operación de git/`gh` si se le pide
  explícitamente en esa sesión, y con el alcance exacto que se le pidió.

## 2. Stack (decisiones cerradas)

- **Backend:** Node.js/Express, acceso a datos y migraciones con Prisma.
- **Servicio de IA:** Python/FastAPI, ONNX Runtime (no PyTorch, para mantener la
  imagen liviana), embeddings con `sentence-transformers` local (384 dimensiones).
- **Frontend:** React.
- **Base de datos:** PostgreSQL con `pgvector`, en un contenedor Docker tanto en
  local como en staging (sin base administrada).
- **IA generativa:** Gemini API, tier pago cubierto por el crédito de Google AI Pro.
- **Staging:** una VM de OCI Always Free que corre el mismo compose de
  `nutria-backend/` más `docker-compose.staging.yml` (HTTPS, sin pgAdmin, sin puerto
  de Postgres, servicio de IA sin ruta pública). Se despliega con cada push a
  `staging` (`cd-staging.yml` por SSH). Detalle en `docs/estrategia-de-despliegue.md`
  y en el README de `nutria-backend`, sección "Staging". Producción sin definir.
- **Auth:** JWT + bcrypt; todo acceso a datos de paciente filtra por
  `nutricionista_id` en middleware — nunca se lee `nutricionista_id` del cliente.
- **docker-compose local es siempre el fallback funcional**: staging usa ese mismo
  compose, así que lo que anda en local anda en staging.

## 3. Prohibiciones transversales

- No inventar reglas clínicas ni datos de validación: el motor de reglas
  deterministico vive en el backend, no en el servicio de IA, y cada regla tiene su
  ficha con origen (E/P/S/V) en `docs/reglas-clinicas.md`. Toda sugerencia de
  IA debe poder ser revisada, editada, aceptada o rechazada por el nutricionista.
- No usar `prisma db push` fuera de prototipado rápido descartable: los cambios de
  esquema van por migración nombrada (ver skill `prisma-migraciones`).
- No enviar datos personales identificables de pacientes al proveedor de IA externo
  (ver skill `servicio-ia-fastapi` para el detalle de qué sí/no se envía).
- No usar datos reales de pacientes en pruebas o demos: solo datos ficticios o
  anonimizados.
- No registrar como "implementado" algo que no tiene prueba o verificación que lo
  respalde.
- **Tests, migraciones y seed se corren siempre dentro del contenedor Docker del
  repo**, nunca con el Node/Python del host. Los comandos viven en el
  `package.json` de la raíz `nutria/` y se ejecutan desde ahí (`npm run
  test:backend`, `npm run db:migrate:dev -- --name ...`, `npm run db:seed`, etc.;
  lista completa en el `README.md` de la raíz). Un `npm test` / `pytest` /
  `npx prisma` suelto en local sirve para iterar, pero no cuenta como
  verificación de una tarea. Corren con `docker compose run --build`: rebuildean
  con cache antes de ejecutar, así que un test o cambio de código nuevo se ve sin
  ningún paso manual (el rebuild cacheado tarda segundos, no reinstala
  dependencias salvo que hayan cambiado). No hay un `docker-compose.dev.yml` ni
  nada por el estilo — solo el compose normal de `nutria-backend/`. La única
  excepción es `docker-compose.staging.yml`, que se usa solo en la VM de staging.
- `npm run up` deja las migraciones al día solo: el `CMD` del backend corre
  `prisma migrate deploy` antes de levantar el server. No hace falta `db:migrate`
  aparte después de un merge, salvo que quieras aplicarlas sin reiniciar el stack.

## 4. Backend (`nutria-backend/`)

Capas: schema Zod → repositorio → servicio → controlador → ruta → test de
aislamiento → docs. Formato de error uniforme en toda la API. Detalle completo en
el skill `backend-express-capas`; migraciones en `prisma-migraciones`; tests en
`pruebas-vitest`; motor de reglas y hallazgos (`src/domain/reglas/`) en
`reglas-clinicas-deterministicas`.

## 5. Servicio de IA (`nutria-ia/`)

FastAPI con contrato Pydantic estricto (parseo estricto de la respuesta del
modelo, sin aceptar texto libre donde se espera una estructura). Prompts
versionados. Trazabilidad de entradas/salidas sin datos personales. No decide
restricciones clínicas — eso es del motor determinístico del backend. Detalle en
el skill `servicio-ia-fastapi`.

## 6. Frontend (`nutria-frontend/`)

Pantallas construidas a partir del diseño en Figma y del alcance específico de la
tarea — lo que no está en el alcance de la tarea se oculta, no se deja a medio
implementar. Cliente HTTP único (sin `fetch` sueltos por componente). Errores de
formulario mostrados por campo. Detalle en el skill `frontend-react-nutria`.

## 7. CI (ver skill `ci-github-actions`)

Cada repo tiene su propio `.github/workflows/ci.yml`, disparado por PR hacia
`development`/`master`, push a `development` y a mano (`workflow_dispatch`). El job
`ci` es el check obligatorio del ruleset de `development` y `master`. La CI no usa
secrets; el CD va en otro workflow (card de staging).
