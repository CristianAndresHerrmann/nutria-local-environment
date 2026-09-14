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

## 2. Stack (decisiones cerradas)

- **Backend:** Node.js/Express, acceso a datos y migraciones con Prisma.
- **Servicio de IA:** Python/FastAPI, ONNX Runtime (no PyTorch, para mantener la
  imagen liviana), embeddings con `sentence-transformers` local (384 dimensiones).
- **Frontend:** React.
- **Base de datos:** PostgreSQL con `pgvector`; en local, contenedor Docker; en
  despliegue, Supabase free tier (solo como Postgres administrado — no se usan
  Auth, PostgREST ni Edge Functions, para preservar portabilidad).
- **IA generativa:** Gemini API, tier pago cubierto por el crédito de Google AI Pro.
- **Infraestructura de despliegue (Incremento 3):** GCP Cloud Run en `us-central1`,
  imágenes construidas localmente/por Actions (Container Scanning API deshabilitada,
  sin Cloud Build). El servicio de IA se despliega privado, con autenticación por
  identity token desde el backend.
- **Auth:** JWT + bcrypt; todo acceso a datos de paciente filtra por
  `nutricionista_id` en middleware — nunca se lee `nutricionista_id` del cliente.
- **docker-compose local es siempre el fallback funcional**; GCP es solo para la
  presentación final ante el tribunal.

## 3. Prohibiciones transversales

- No inventar reglas clínicas ni datos de validación: el motor de reglas
  deterministico vive en el backend, no en el servicio de IA. Toda sugerencia de
  IA debe poder ser revisada, editada, aceptada o rechazada por el nutricionista.
- No usar `prisma db push` fuera de prototipado rápido descartable: los cambios de
  esquema van por migración nombrada (ver skill `prisma-migraciones`).
- No enviar datos personales identificables de pacientes al proveedor de IA externo
  (ver skill `servicio-ia-fastapi` para el detalle de qué sí/no se envía).
- No usar datos reales de pacientes en pruebas o demos: solo datos ficticios o
  anonimizados.
- No registrar como "implementado" algo que no tiene prueba o verificación que lo
  respalde.

## 4. Backend (`nutria-backend/`)

Capas: schema Zod → repositorio → servicio → controlador → ruta → test de
aislamiento → docs. Formato de error uniforme en toda la API. Detalle completo en
el skill `backend-express-capas`; migraciones en `prisma-migraciones`; tests en
`pruebas-vitest`.

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
`development`/`master` y push a `development`. Sin secretos en el Incremento 1.
