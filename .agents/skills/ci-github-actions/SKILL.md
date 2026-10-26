---
name: ci-github-actions
description: Estructura de los workflows de CI y CD a staging de NutrIA, cómo agregar un paso nuevo y qué hacer cuando un check o un despliegue falla. Usar al tocar archivos dentro de .github/workflows.
---

# CI/CD con GitHub Actions — NutrIA

Cada uno de los tres repos tiene su propio `.github/workflows/ci.yml` y
`cd-staging.yml` (no hay workflows compartidos entre repos).

- `ci.yml`: `pull_request` hacia `development`/`staging`/`master`, `push` a
  `development`, `workflow_dispatch` y `workflow_call` (lo reutiliza el CD). No
  despliega nada y no usa secrets.
- `cd-staging.yml`: `push` a `staging` y `workflow_dispatch`. Corre el CI y, si pasa,
  despliega en la VM de staging (ver "CD a staging").

`nutria-local-environment` (la raíz `nutria/`) no lleva CI.

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
6. `docker build` (sin push) del target por defecto (`dev`) y del `prod` (staging)

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

## CD a staging

`cd-staging.yml`, mismo esqueleto en los tres repos; solo cambian el servicio y el
smoke test:

1. Job `ci`: `uses: ./.github/workflows/ci.yml`. Si falla, no se despliega.
2. Job `deploy` (`needs: ci`, `environment: staging`): instala la clave y el
   `known_hosts` desde los secrets y corre
   `ssh $STAGING_SSH_USER@$STAGING_HOST <backend|frontend|ia>`.
3. Smoke test por HTTPS con reintentos (~3 min): `/health` y `/health/ia` del backend,
   la página del frontend. La IA no tiene ruta pública: se verifica por `/health/ia`.

En la VM, la clave de despliegue está en `authorized_keys` con
`restrict,command="bash …/nutria-backend/scripts/deploy-staging.sh"`: pida lo que
pida, solo corre ese script, que valida el servicio, toma un lock, deja el repo igual
a `origin/staging` y rebuildea con
`docker compose -f docker-compose.yml -f docker-compose.staging.yml`. El del backend
levanta todo el stack (el compose y Traefik viven ahí).

- `concurrency: cd-staging` sin cancelar: un despliegue a medias puede dejar la VM
  en un estado intermedio.
- Si un despliegue necesita otro comando en la VM, se agrega al script (con su
  validación), no al workflow: la clave no puede ejecutar nada más.
- Detalle operativo (variables de la VM, primer arranque, comandos): README de
  `nutria-backend`, sección "Staging".

## Protección de ramas

Ruleset por repo sobre `development`, `staging` y `master`, con bypass list vacía: bloquea
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

El CD usa el Environment `staging` de cada repo (restringido a la rama `staging`),
nunca secrets a nivel repo ni en `ci.yml`:

| Nombre | Tipo |
|---|---|
| `STAGING_SSH_KEY` (clave privada de despliegue) | secret |
| `STAGING_SSH_KNOWN_HOSTS` (`<IP> ssh-ed25519 …`) | secret |
| `STAGING_HOST`, `STAGING_SSH_USER`, `STAGING_DOMAIN` | variables |

En los `run:` se pasan por `env:`, no interpolados en el script. Los secrets de la
aplicación (`JWT_SECRET`, contraseñas) viven en los `.env` de la VM, no en GitHub.

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

## Cuando un despliegue falla

1. **Falla el job `ci`:** es el mismo caso de arriba; no se desplegó nada.
2. **Falla el SSH** (`Permission denied`, `Host key verification failed`): revisar
   los secrets del Environment y la línea de `authorized_keys` en la VM. Si la VM se
   recreó, cambió su clave de host y hay que actualizar `STAGING_SSH_KNOWN_HOSTS`;
   nunca desactivar `StrictHostKeyChecking`.
3. **Falla el script o el smoke test:** entrar a la VM con la clave personal y mirar
   `dcs ps` y `dcs logs <servicio>` (alias del README de `nutria-backend`). Un
   certificado que todavía no salió se ve en `dcs logs proxy`.
4. Corregir con un PR nuevo y volver a desplegar; no editar archivos a mano en la VM
   (el siguiente despliegue los pisa con `git reset --hard`).
