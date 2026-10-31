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
- **`docs/reglas-clinicas.md`**: catálogo de reglas del motor determinístico, cada una
  con su origen.
- **`.agents/skills/`**: procedimientos paso a paso para trabajar en el proyecto.

## Requisitos

- Git, con acceso a los tres repos (son privados).
- Docker Desktop (Compose v2.23 o superior; `npm run dev` lo necesita).
- Node.js, solo para lanzar los comandos con `npm run` — el código nunca corre con
  el Node del host. No hace falta `npm install` en esta carpeta.
- El puerto 80 del host libre: lo usa el proxy del stack.

## Primer arranque

Todo desde esta carpeta (`nutria/`).

**1. Clonar los tres repos** como carpetas hermanas, con estos nombres exactos (el
`docker-compose.yml` del backend los busca por ruta relativa). El repo remoto del
frontend se llama `nutria-frontned`; la carpeta local tiene que ser
`nutria-frontend`:

```bash
git clone https://github.com/CristianAndresHerrmann/nutria-backend.git nutria-backend
git clone https://github.com/CristianAndresHerrmann/nutria-frontned.git nutria-frontend
git clone https://github.com/CristianAndresHerrmann/nutria-ia.git nutria-ia
```

**2. Crear los `.env`** a partir de los `.env.example`:

```bash
cp nutria-backend/.env.example nutria-backend/.env
cp nutria-frontend/.env.example nutria-frontend/.env
cp nutria-ia/.env.example nutria-ia/.env
```

En `nutria-backend/.env` completar `POSTGRES_USER`, `POSTGRES_PASSWORD`,
`POSTGRES_DB` y `JWT_SECRET` con valores cualquiera para uso local. Los `.env` del
frontend y de IA sirven tal cual vienen.

**3. Registrar los dominios locales** en el archivo `hosts`, una sola vez por
máquina. En Windows es `C:\Windows\System32\drivers\etc\hosts` (editar como
administrador); en Linux/macOS, `/etc/hosts`:

```
127.0.0.1 local.nutria.com
127.0.0.1 api.local.nutria.com
127.0.0.1 ia.local.nutria.com
127.0.0.1 swagger.local.nutria.com
127.0.0.1 db.local.nutria.com
```

**4. Levantar el stack:**

```bash
npm run up
```

La primera vez construye las imágenes (tarda unos minutos). Las migraciones se
aplican solas al arrancar el backend.

**5. Cargar los datos de prueba:**

```bash
npm run db:seed
```

Carga el catálogo común (grupos, alimentos, patologías y recomendaciones de
ejemplo) y dos nutricionistas, cada uno con pacientes ficticios ya cargados
(condiciones, consultas y mediciones), recetas, y alimentos y recomendaciones
privados. Al terminar imprime los emails y la contraseña para iniciar sesión. Todo
es ficticio.

Opcional, para usar el panel de cuentas (`/admin`): completar `ADMIN_EMAIL` y
`ADMIN_PASSWORD` en `nutria-backend/.env` y correr `npm run db:create-admin`. El
administrador entra con el mismo login.

**6. Verificar:**

| URL | Qué debería verse |
|---|---|
| http://local.nutria.com | Login de la aplicación (entrar con las credenciales del paso 5) |
| http://api.local.nutria.com/health | `{"status":"ok", ...}` del backend |
| http://ia.local.nutria.com/health | `{"status":"ok", ...}` del servicio de IA |
| http://swagger.local.nutria.com | Documentación interactiva de la API |
| http://db.local.nutria.com | pgAdmin, con la base del stack ya registrada |

**7. Correr las pruebas:** `npm run test:backend` y `npm run test:frontend` (ver
[Tests](#tests)).

Para bajar el stack: `npm run down` (los datos quedan en el volumen de Postgres).

## Comandos

Todos se corren **desde esta carpeta** (`nutria/`), y todos ejecutan dentro de los
contenedores Docker de cada repo. Regla del proyecto: un `npm test`, `pytest` o
`npx prisma` suelto en el host sirve para iterar, pero no cuenta como verificación
de una tarea.

### Stack

| Comando | Qué hace |
|---|---|
| `npm run up` | Levanta todo el stack (`proxy`, `frontend`, `backend`, `ia`, `postgres`, `pgadmin`) rebuildeando imágenes. El frontend queda con hot reload, también al reiniciarlo desde Docker Desktop. |
| `npm run dev` | Levanta el stack en primer plano y agrega Compose Watch para reiniciar el backend al editar `src` y reconstruir imágenes al cambiar dependencias. El hot reload del frontend funciona también sin este comando. Ctrl+C detiene el stack. |
| `npm run down` | Baja el stack. Los datos de Postgres quedan en el volumen. |
| `npm run ps` | Estado de los contenedores. |
| `npm run logs` | Logs en vivo de todo. Para un servicio: `npm run logs -- backend`. |
| `npm run build` | Rebuildea las imágenes sin levantar nada. Para una sola: `npm run build -- backend`. |

### Tests

| Comando | Qué hace |
|---|---|
| `npm run test:backend` | Vitest del backend. |
| `npm run test:backend:integration` | Integración del backend contra Postgres real. Levanta `postgres`, recrea solo `nutria_test` con las migraciones y corre los tests en serie, con fixtures ficticias. |
| `npm run test:backend:coverage` | Ídem + cobertura: tabla en terminal y reporte en `nutria-backend/coverage/index.html` (y `lcov.info`). |
| `npm run test:ia` | `pytest` del servicio de IA, en el servicio `ia-test` del compose (etapa `test` del Dockerfile de `nutria-ia`, con `requirements-dev.txt`). |
| `npm run test:frontend` | Vitest del frontend. |

La integración usa las credenciales y el servidor de `DATABASE_URL`, cambiando la
base por `nutria_test`. Esa base queda reservada para pruebas y se vacía en cada
corrida; la base de desarrollo no se modifica. Opcionalmente se puede definir
`TEST_DATABASE_URL` en `nutria-backend/.env`: debe apuntar a `nutria_test` con el
schema `public`, o el comando falla antes del reset. Los archivos `*.int.test.js`
se excluyen de la suite unitaria. No requiere levantar IA ni frontend.

### Contrato backend ↔ IA

El OpenAPI de IA se versiona en `nutria-ia/contracts/openapi.json`, con una copia
en `nutria-backend/src/schemas/fixtures/ia.openapi.json`. `test:ia` comprueba que
FastAPI sigue publicando ese contrato; `test:backend` compara los enums de Prisma,
la propuesta validada por Zod y los topes del catálogo y textos de contexto con
su copia. Cada repo puede ejecutar su CI sin clonar el otro.

Cuando un cambio intencional afecte el contrato, revisar su compatibilidad y
ejecutar desde esta carpeta:

```bash
npm run contract:ia:update
npm run test:ia
npm run test:backend
```

El primer comando genera ambas copias desde FastAPI dentro de Docker, sin
levantar servidores ni llamar al proveedor. Las copias se actualizan a propósito:
las suites no las regeneran ni ocultan una divergencia. Al cambiar el contrato,
ambos repos deben incorporar la actualización coordinada antes de desplegar.

### Base de datos

Todos levantan `postgres` si no está y esperan a que responda antes de conectar.

Con el stack levantado (`npm run up`), la base también se puede ver desde el navegador en http://db.local.nutria.com (pgAdmin, sin login; ver README de `nutria-backend/` para los virtual hosts).

| Comando | Qué hace |
|---|---|
| `npm run db:migrate` | Aplica las migraciones pendientes (`prisma migrate deploy`). No crea migraciones. |
| `npm run db:migrate:dev -- --name <verbo_entidad_cambio>` | Genera una migración nueva a partir de `schema.prisma` y la aplica. Queda en `nutria-backend/prisma/migrations/`. Ver skill `prisma-migraciones` para el nombre. |
| `npm run db:seed` | Corre `nutria-backend/scripts/seed.js` (datos ficticios). |
| `npm run db:create-admin` | Crea o pone al día la cuenta de administrador con `ADMIN_EMAIL` y `ADMIN_PASSWORD` del `.env` de `nutria-backend/`. Idempotente. |
| `npm run db:reset` | **Destructivo.** Borra la base local, aplica todas las migraciones desde cero y corre el seed. Es la "verificación desde cero" del skill `prisma-migraciones`. |
| `npm run db:studio` | Prisma Studio en http://localhost:5555 (Ctrl+C para cerrarlo). |
| `npm run db:shell` | `psql` conectado a la base del stack. |

### Reconstruir la base

- **Desde cero:** `npm run db:reset`. Borra todo, aplica las migraciones desde la
  primera y corre el seed. Sirve para descartar los datos de prueba o después de
  cambiar a una rama con otras migraciones.
- **Si el backend estaba levantado**, reiniciarlo después del reset
  (`docker restart nutria-backend-1`). Prisma guarda en caché los tipos de la base y,
  hasta reiniciar, responde `500` ("cache lookup failed for type…").
- **Solo el seed:** `npm run db:seed` se puede volver a correr sobre una base con
  datos. Actualiza el catálogo común sin duplicarlo y recrea los pacientes, las
  recetas y las recomendaciones privadas de los dos nutricionistas del seed: lo que
  se haya cargado a mano en esas dos cuentas se pierde, salvo los alimentos
  privados. Las demás cuentas (las del panel de administración y la del administrador)
  no se tocan. `db:reset` sí las borra: después de un reset hay que volver a correr
  `npm run db:create-admin`.

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
testear**. Se usa el mismo `docker-compose.yml`. El frontend lee el código desde
sus mounts de solo lectura; las dependencias y la ejecución de las pruebas
permanecen dentro del contenedor.

Dos comandos puntuales (`db:migrate:dev`, que genera un archivo de migración, y
`test:backend:coverage`, que genera un reporte HTML) agregan un `-v` al comando
para que ese archivo quede en el host en vez de perderse con el contenedor
efímero — nada permanente, solo para esa corrida.

Las imágenes de Node instalan con `npm ci`, respetando el lockfile. El backend
construye y ejecuta sus comandos como `node` (UID/GID 1000), incluidas las pruebas
y Prisma. `test:backend:coverage` elimina el reporte generado anterior y recrea la
carpeta de salida con el usuario del host; los tests y los reportes se ejecutan
dentro del contenedor. Esto evita reutilizar archivos de cobertura escritos por
root o que Docker cree el directorio como root cuando todavía no existe. En Linux,
los directorios montados (`prisma/` y `coverage/`) deben permitir escritura al UID
1000. Si un reporte anterior creado por root no permite limpiarlo, hay que corregir
su propietario antes de ejecutar el comando.

`npm run up` también rebuildea (por eso el `--build` en su script) y además dejás
las migraciones al día: el `CMD` del backend corre `prisma migrate deploy` antes
de arrancar el server, así que un `git pull`/merge que trae una migración nueva
no requiere ningún paso manual — `npm run up` sola te deja el schema
sincronizado.

## Ver cambios sin rebuildear

El **frontend tiene hot reload al iniciar desde Docker Desktop**. Una vez creados
los contenedores con `npm run up`, podés detener/iniciar el stack desde la interfaz
de Docker Desktop: guardás un archivo y Vite actualiza `http://local.nutria.com`.
No requiere una terminal abierta ni ejecutar Compose Watch.

El compose monta `src/`, `public/`, `index.html` y `vite.config.js` del frontend en
solo lectura. Vite consulta cambios cada 500 ms (`WATCH_POLLING=true` en el servicio
local), para detectar también las ediciones de Windows/OneDrive. `node_modules`
queda dentro de la imagen, con las dependencias de Linux. Staging elimina estos
mounts y el polling mediante su override y sirve el build estático de nginx.

Si ya había un contenedor creado antes de esta configuración, recrearlo una vez:

```bash
docker compose -f nutria-backend/docker-compose.yml up -d --build --no-deps frontend
```

Esto lo incorpora al contenedor; reiniciar uno viejo desde Docker Desktop no
aplica cambios en el compose. Después alcanza con los controles de Docker Desktop.

Para observar además el **backend y las dependencias**, `npm run dev` agrega
Compose Watch (bloque `develop.watch` del `docker-compose.yml`):

| Qué cambia | Qué pasa |
|---|---|
| `nutria-frontend/src`, `public/` o `index.html` | Hot reload de Vite por los mounts, incluso sin Compose Watch. |
| `nutria-frontend/vite.config.js` | Vite reinicia automáticamente su servidor de desarrollo. |
| `nutria-backend/src` | Se copia y se reinicia el contenedor del backend (unos segundos). |
| `package.json`/`package-lock.json` de cualquiera de los dos o `nutria-backend/prisma/` | Rebuild automático de esa imagen con Compose Watch activo. |
| `nutria-ia` | No se mira: sigue necesitando `npm run up`. |

Al usar solo Docker Desktop, los cambios de dependencias requieren reconstruir la
imagen; los cambios de variables de entorno o del compose requieren recrear el
contenedor. Los archivos de código del frontend no necesitan ninguno de esos pasos.

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
