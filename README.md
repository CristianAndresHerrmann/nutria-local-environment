# nutria — tooling transversal

Este repo chico versiona lo que es común a los tres repos de NutrIA: reglas para
asistentes de IA (`AGENTS.md`, `.agents/skills/`), decisiones técnicas
(`docs/decisiones/`) y los comandos de este archivo. Los tres repos de código se
clonan como carpetas hermanas dentro de esta:

```
nutria/
├── nutria-backend/    (Node/Express + Prisma; acá vive el docker-compose del stack)
├── nutria-frontend/   (React + Vite)
└── nutria-ia/         (Python/FastAPI)
```

## Qué va en cada archivo de documentación

- **`README.md`** (este y el de cada repo): información del repositorio — para qué
  es, cómo se estructura, cómo se levanta, qué variables y comandos tiene. Nada de
  detalle técnico de funcionalidades (endpoints, contratos, comportamiento de una
  pantalla); a lo sumo una frase de alto nivel, y solo si hace falta para ubicarse.
- **`CHANGELOG.md`** de cada repo: qué se agregó, cambió o arregló en cada
  iteración, con el detalle funcional.
- **`docs/decisiones/`**: decisiones técnicas con su fundamento (una por archivo,
  a partir de `_plantilla.md`).
- **`.agents/skills/`**: procedimientos paso a paso para trabajar en el proyecto.

## Requisitos

- Docker Desktop (Compose v2.20 o superior).
- Node.js, solo para lanzar los comandos con `npm run` — el código nunca corre con
  el Node del host.
- El `.env` de cada repo creado a partir de su `.env.example` (ver README de
  `nutria-backend/`).

## Comandos

Todos se corren **desde esta carpeta** (`nutria/`), y todos ejecutan dentro de los
contenedores Docker de cada repo. Regla del proyecto: un `npm test`, `pytest` o
`npx prisma` suelto en el host sirve para iterar, pero no cuenta como verificación
de una tarea.

### Stack

| Comando | Qué hace |
|---|---|
| `npm run up` | Levanta todo el stack (`proxy`, `frontend`, `backend`, `ia`, `postgres`) rebuildeando imágenes. |
| `npm run down` | Baja el stack. Los datos de Postgres quedan en el volumen. |
| `npm run ps` | Estado de los contenedores. |
| `npm run logs` | Logs en vivo de todo. Para un servicio: `npm run logs -- backend`. |
| `npm run build` | Rebuildea las imágenes sin levantar nada. Para una sola: `npm run build -- backend`. |

### Tests

| Comando | Qué hace |
|---|---|
| `npm run test:backend` | Vitest del backend. |
| `npm run test:backend:coverage` | Ídem + cobertura: tabla en terminal y reporte en `nutria-backend/coverage/index.html` (y `lcov.info`). |
| `npm run test:ia` | `pytest` del servicio de IA. Hoy falla con `pytest: not found` porque ese repo todavía no tiene tests ni `pytest` en `requirements.txt`. |
| `npm run test:frontend` | Vitest del frontend. |

### Base de datos

Todos levantan `postgres` si no está y esperan a que responda antes de conectar.

| Comando | Qué hace |
|---|---|
| `npm run db:migrate` | Aplica las migraciones pendientes (`prisma migrate deploy`). No crea migraciones. |
| `npm run db:migrate:dev -- --name <verbo_entidad_cambio>` | Genera una migración nueva a partir de `schema.prisma` y la aplica. Queda en `nutria-backend/prisma/migrations/`. Ver skill `prisma-migraciones` para el nombre. |
| `npm run db:seed` | Corre `nutria-backend/scripts/seed.js` (datos ficticios). |
| `npm run db:reset` | **Destructivo.** Borra la base local, aplica todas las migraciones desde cero y corre el seed. Es la "verificación desde cero" del skill `prisma-migraciones`. |
| `npm run db:studio` | Prisma Studio en http://localhost:5555 (Ctrl+C para cerrarlo). |
| `npm run db:shell` | `psql` conectado a la base del stack. |

### Shell dentro de un contenedor

`npm run shell:backend`, `npm run shell:ia`, `npm run shell:frontend` — requieren
el stack levantado (`npm run up`).

## Cómo evitan el rebuild manual

Los comandos de tests y base de datos usan `docker compose run --build`: cada
corrida rebuildea la imagen primero, con cache de capas. Un test nuevo o un
cambio de código solo invalida la capa que copia el código fuente — Docker no
reinstala dependencias salvo que `package.json`/`package-lock.json` (o
`requirements.txt` en `ia`) hayan cambiado — así que en la práctica el rebuild
tarda segundos y **nunca hay que acordarse de rebuildear a mano antes de
testear**. No hay ningún archivo de compose extra para esto ni código montado
de forma permanente: es el `docker-compose.yml` de siempre con un flag.

Dos comandos puntuales (`db:migrate:dev`, que genera un archivo de migración, y
`test:backend:coverage`, que genera un reporte HTML) agregan un `-v` al comando
para que ese archivo quede en el host en vez de perderse con el contenedor
efímero — nada permanente, solo para esa corrida.

`npm run up` también rebuildea (por eso el `--build` en su script) y además dejás
las migraciones al día: el `CMD` del backend corre `prisma migrate deploy` antes
de arrancar el server, así que un `git pull`/merge que trae una migración nueva
no requiere ningún paso manual — `npm run up` sola te deja el schema
sincronizado.

## Skills para Claude Code

Los skills viven en `.agents/skills/`; Claude Code los busca en `.claude/skills/`,
que es un symlink local (no versionado). Si no existe, crearlo una vez:

```powershell
# Windows (PowerShell como administrador, o con Modo de desarrollador activado)
New-Item -ItemType SymbolicLink -Path .claude\skills -Target ..\.agents\skills
```

```bash
# Linux / macOS / Git Bash
ln -s ../.agents/skills .claude/skills
```
