---
name: ci-github-actions
description: Estructura de los workflows de CI de NutrIA, cómo agregar un paso nuevo y qué hacer cuando un check falla. Usar al tocar archivos dentro de .github/workflows.
---

# CI con GitHub Actions — NutrIA

Cada uno de los tres repos tiene su propio `.github/workflows/ci.yml` (no hay
workflow compartido). Disparadores: `pull_request` hacia `development`/`master`,
`push` a `development` y `workflow_dispatch`. Solo integración continua: no
despliega nada y no usa secrets. `nutria-local-environment` (la raíz `nutria/`) no
lleva CI.

## Estructura por repo

Un solo job por repo, llamado `ci`, en `ubuntu-24.04` con `timeout-minutes: 10`.

**`nutria-backend/ci.yml`** — servicio Postgres `pgvector/pgvector:pg16` con
credenciales efímeras escritas en el workflow y `DATABASE_URL` a nivel job.
1. `npm ci`
2. `npx prisma validate` y `npx prisma generate`
3. `npx prisma migrate deploy` sobre la base vacía (las migraciones aplican desde cero)
4. `npm run seed` (corre sobre una base recién migrada, DA-24)
5. `npm test`
6. `npm audit --audit-level=high`
7. `docker build` (sin push)

Sin lint en el backend (DA-24).

**`nutria-frontend/ci.yml`**
1. `npm ci`
2. `npm run lint` (oxlint)
3. `npm test`
4. `npm run build`
5. `npm audit --audit-level=high`
6. `docker build` (sin push)

**`nutria-ia/ci.yml`**
1. `pip install -r requirements.txt -r requirements-dev.txt`
2. `ruff check`
3. `pytest`
4. `pip-audit -r requirements.txt`
5. `docker build` (target por defecto `runtime`, sin dependencias de desarrollo) +
   verificación de que pesa menos de 0,5 GB (500 000 000 bytes; sostiene la
   decisión de usar ONNX Runtime en vez de PyTorch)

`requirements-dev.txt` no entra en la imagen de runtime: lo instala la etapa `test`
del Dockerfile, que es la que usa el servicio `ia-test` del compose
(`npm run test:ia`).

Versiones: Node 22 y Python 3.12, las mismas que los Dockerfiles. Caché de
`npm`/`pip` vía `setup-node`/`setup-python`. Objetivo: menos de 5 minutos por
workflow. Cómo reproducir cada paso en local: sección "Integración continua" del
README de cada repo.

## Reglas del workflow

- **Acciones fijadas por SHA completo con el tag en un comentario**
  (`actions/checkout@<sha40> # v7.0.1`), nunca `@latest` ni `@v7`. Para subir de
  versión, resolver el SHA del tag en la API de GitHub y actualizar los tres repos.
  Preferir una release con al menos dos semanas de publicada.
- Solo acciones oficiales de `actions/*`. Lo demás (docker build, chequeo de tamaño)
  va como `run:`.
- `permissions: contents: read` a nivel workflow y `persist-credentials: false` en
  el checkout.
- **No renombrar el job `ci`**: es el check obligatorio del ruleset.
- **No agregar filtros `paths:`/`paths-ignore:`** al trigger: un PR que no dispara
  el workflow queda bloqueado esperando un check que nunca reporta.
- No usar `pull_request_target`: los repos son públicos y ejecutaría código de un
  fork con permisos del repo.

## Protección de ramas

Ruleset por repo sobre `development` y `master`, con bypass list vacía: bloquea
borrado y force push, exige PR (0 aprobaciones: un solo desarrollador) y exige el
check `ci` en verde (sin "require branches to be up to date"). El check solo aparece
en el buscador del ruleset después de que el workflow corrió al menos una vez.

## Secrets y variables

La CI no usa ningún secret ni variable de GitHub, y así se mantiene:

- Credenciales de Postgres y `DATABASE_URL`: efímeras, escritas en el workflow.
- `JWT_SECRET` de los tests: lo fija `vitest.config.js`.
- `VITE_API_URL`: el build no la necesita. Si se agrega, como `env:` plano (termina
  en el bundle público, no es secreta).
- Proveedor de IA: ningún test lo llama. Un test que lo necesite usa un doble, nunca
  una API key.
- Los `.env` no se versionan ni se crean en la CI; `docker build` no los lee.

Si un paso parece necesitar un secret, primero confirmar con quien inicia la sesión
si corresponde a esta etapa — no agregar secrets al workflow sin decisión explícita.

El CD vive en otro workflow (card "Desplegar staging bajo demanda"), nunca en
`ci.yml`: sus secrets y variables van en el Environment `staging` de GitHub, la
autenticación con GCP es por Workload Identity Federation (sin claves JSON;
`id-token: write` solo en ese job), y los identificadores de GCP son variables, no
secrets.

## Cómo agregar un paso

1. Agregarlo en el job `ci` del `ci.yml` de ese repo únicamente.
2. Si el paso necesita una herramienta, fijar la versión explícitamente (no `latest`).
3. Documentarlo en la tabla "Integración continua" del README del repo, con el
   comando para reproducirlo en local dentro del contenedor.
4. Probarlo en un PR real antes de darlo por cerrado; no alcanza con que el YAML
   sea válido (`actionlint` ayuda, pero no lo reemplaza).

Integración contra Postgres, E2E y carga (DA-24) se suman al workflow cuando
existan, cada una en su propia tarea.

## Cuando un check falla

1. Leer el log del step que rompió, no solo el resumen rojo.
2. Reproducir el mismo comando en local, dentro del contenedor (tabla del README),
   antes de tocar el workflow — la mayoría de las fallas son del código, no de la
   config de CI.
3. Si es una falla de infraestructura del propio workflow (servicio Postgres no
   levanta, dependencia no resuelve), revisar versión fijada y logs del servicio,
   no agregar reintentos como parche.
4. Si falla `npm audit` o `pip-audit`, actualizar la dependencia en su propia tarea
   o PR; no bajar el `--audit-level` ni sacar el paso para destrabar un merge.
5. No mergear con el check en rojo. No usar `[skip ci]` para esquivar una falla.
