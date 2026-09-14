---
name: ci-github-actions
description: Estructura de los workflows de CI de NutrIA, cómo agregar un paso nuevo y qué hacer cuando un check falla. Usar al tocar archivos dentro de .github/workflows.
---

# CI con GitHub Actions — NutrIA

Cada uno de los tres repos tiene su propio `.github/workflows/ci.yml`, disparado
por `pull_request` hacia `development`/`master` y `push` a `development`.
Incremento 1: solo integración continua, sin despliegue y sin secretos.

## Estructura por repo

**`nutria-backend/ci.yml`**
1. Servicio Postgres `pgvector/pgvector:pg16`.
2. `npm ci`
3. `npx prisma validate`
4. `npx prisma migrate deploy`
5. `npm test`
6. `docker build` (sin push)

**`nutria-ia/ci.yml`**
1. `pip install -r requirements.txt`
2. `ruff check`
3. `pytest`
4. `docker build` + verificación de que la imagen final pesa menos de 0.5 GB
   (sostiene la decisión de usar ONNX Runtime en vez de PyTorch)

**`nutria-frontend/ci.yml`**
1. `npm ci`
2. lint
3. `npm run build`
4. tests, si existen

Versiones fijadas: Node 22, Python 3.12 (ajustar si T2 definió otra cosa, y
mantenerlas iguales entre Dockerfile y CI). Usar caché de `npm`/`pip`; objetivo
menor a 5 minutos por workflow.

## Cómo agregar un paso

1. Agregarlo en el job correspondiente del `ci.yml` de ese repo únicamente — no
   hay un workflow compartido entre repos.
2. Si el paso necesita una versión de herramienta, fijarla explícitamente (no usar
   `latest`).
3. Si el paso necesita un secreto, primero confirmar si corresponde a esta etapa
   (Incremento 1 no usa secretos) o si hay que escalarlo — no agregar secretos al
   workflow sin decisión explícita.
4. Probarlo en un PR real antes de darlo por cerrado; no alcanza con que el YAML
   sea válido.

## Cuando un check falla

1. Leer el log del step que rompió, no solo el resumen rojo.
2. Reproducir el mismo comando en local, dentro del repo correspondiente
   (`npm test`, `pytest`, etc.) antes de tocar el workflow — la mayoría de las
   fallas son del código, no de la config de CI.
3. Si es una falla de infraestructura del propio workflow (servicio Postgres no
   levanta, dependencia no resuelve), revisar versión fijada y logs del servicio,
   no agregar reintentos como parche.
4. No mergear con el check en rojo. No usar `[skip ci]` para esquivar una falla.

## Diferido (no forma parte del Incremento 1)

- Keep-alive de Supabase (cron cada 3 días, secreto `SUPABASE_DB_URL`).
- Fast-forward/PR automatizado de `development` a `staging`.
- CD a Cloud Run desde `staging` (Incremento 3), con Workload Identity Federation
  — sin claves JSON.
