# Estrategia de despliegue a costo cero

**Fecha:** 21/10/2026
**Estado:** decidida para staging (21/10/2026) y sin implementar. Producción queda sin
definir.
**Fuente de la decisión:** cards de Trello "Desplegar staging bajo demanda (Cloud Run +
Supabase)" y "T12 — Configurar CI/CD con GitHub Actions en los tres repos", las dos en
Backlog. Este documento resume esa decisión y agrega el detalle técnico necesario para
implementarla. Si algo no coincide, mandan las cards.

## 1. Cuándo aplica

| Etapa | Entorno |
|---|---|
| Incremento 2 (en curso) | Solo local, con `npm run up` (docker-compose). |
| Incremento 3 | Solo local hasta terminarlo. Todavía no tiene tareas definidas. |
| Staging | Se implementa cuando el servicio de generación de planes exista (card de staging). Depende del CD (en la card de staging) y de la rama `staging`. |
| Producción | Se define cuando el sistema esté implementado. Fuera del alcance actual. |

Desde el 25/10/2026, T12 es solo CI (checks en cada PR, sin secrets) y el CD pasó a la
card de staging. La CI es independiente del despliegue.

docker-compose local sigue siendo el entorno de respaldo que siempre funciona (riesgo 13
del Plan de Proyecto).

## 2. Antecedentes

- **Plan de Proyecto:** despliegue en "servicios gratuitos o de bajo costo". El riesgo 13
  pide mantener el sistema portable con Docker y, como contingencia, presentar en local.
- **Informe de Avances 2:** DA-14 dejaba la plataforma "en evaluación". Las
  alternativas eran infraestructura propia, OCI, GCP y servicios administrados
  similares. La card de staging resuelve DA-14 **para staging**, y eso se informa en el
  próximo informe de avance.
- **`AGENTS.md` §2:** Cloud Run en `us-central1`, imágenes construidas por Actions (sin
  Cloud Build), servicio de IA privado con identity token y Supabase usado solo como
  Postgres. Es coherente con la card.

## 3. Staging bajo demanda

Staging queda desplegado de forma permanente, pero sin consumo mientras no se usa. Se
activa con aviso previo, para mostrar avances o para que la nutricionista pruebe algo.

```
Navegador
   │ HTTPS
   ├──► Frontend — Cloud Run público (*.run.app)
   └──► Backend Express — Cloud Run público (*.run.app)
            │  identity token (cuenta de servicio del backend)
            ├──► Servicio de IA FastAPI — Cloud Run privado
            │        └──► Gemini API (sin datos identificatorios)
            └──► Postgres — Supabase free, vía pooler
```

| Componente | Plataforma y configuración |
|---|---|
| Backend, IA y frontend | Cloud Run en `us-central1`, `min-instances=0` (sin costo en reposo) y `max-instances=1`. |
| Servicio de IA | Privado (`--no-allow-unauthenticated`). Solo lo invoca el backend, con identity token. |
| Imágenes | Artifact Registry en `us-central1`, tag = SHA del commit, con una política de limpieza que conserva solo las últimas versiones. |
| Base de datos | Supabase free, solo como Postgres (sin Auth, PostgREST ni Edge Functions). El corpus del RAG va en una base separada de la operativa (DA-13). |
| Build y despliegue | GitHub Actions (`cd-staging.yml` en cada repo), disparado por push a `staging` o a mano (`workflow_dispatch`). Autenticación con GCP sin claves JSON. |
| Variables y secretos | Se definen al ejecutar la tarea y se cargan en el Environment `staging` de GitHub. En el README de cada repo se documentan solo los nombres. |
| Dominio | Los que da Cloud Run por defecto (`*.run.app`). |

### Costo esperado

Cero, salvo unos centavos por el almacenamiento de imágenes. Los valores de esta tabla
son de referencia: **hay que verificar las cuotas vigentes al implementar.** El primer
mes de costo real se registra como parte del criterio de cierre de la card.

| Concepto | Cuota sin costo (referencia) | Nota |
|---|---|---|
| Cloud Run | ~2 M solicitudes/mes, ~180 000 vCPU-s, ~360 000 GiB-s | Con escala a cero y uso esporádico, lejos del límite. |
| Artifact Registry | ~0,5 GB de almacenamiento | Las tres imágenes (la de IA con el modelo ONNX) pueden pasarse. El excedente cuesta ~USD 0,10 por GB-mes. |
| Supabase free | 500 MB de base; se pausa tras ~7 días de baja actividad | Datos ficticios del seed. |
| GitHub Actions | Ilimitado en repos públicos | — |
| Gemini API | Tier pago | Según `AGENTS.md`, lo cubre el crédito de Google AI Pro. |

Cloud Run exige una cuenta de facturación aunque el uso no pase de la cuota.

### Barreras contra gastos imprevistos

1. **Alerta de presupuesto** en GCP (por ejemplo, USD 1, con avisos al 50, 90 y 100 %).
   La alerta avisa pero no corta el gasto.
2. **`max-instances=1`** en los tres servicios.
3. **Servicio de IA privado.** Sin identity token, Cloud Run responde 403 antes de que el
   pedido llegue al código o a Gemini.
4. **Cuota de solicitudes** en la clave de Gemini, configurada desde la consola de
   Google Cloud.
5. **Limpieza en Artifact Registry**: conservar 2 versiones por imagen y borrar las que
   no tienen tag.
6. **Nunca dejar `min-instances` mayor a 0.** Una instancia siempre encendida es lo único
   que factura aunque no haya tráfico.

## 4. Operación bajo demanda (antes de cada sesión)

Procedimiento de la card. Se completa con los comandos reales cuando se implemente y se
prueba una vez de punta a punta (criterio de cierre).

1. **Reanudar Supabase** si está pausado. No hay keep-alive: la pausa es aceptable
   porque staging se usa de forma esporádica.
2. **Ejecutar el CD manual** (`workflow_dispatch`) para desplegar la versión
   vigente de `staging`.
3. **Recargar el seed** de pacientes ficticios.
4. **Hacer un pedido de calentamiento** (login y una generación) para que el backend, el
   servicio de IA y la conexión a la base salgan del arranque en frío antes de la
   sesión.

Reglas del entorno:
- Solo datos ficticios. Avisar explícitamente a la nutricionista que no cargue pacientes
  reales.
- Aviso visible en la interfaz de que es un entorno de prueba.
- Acceso solo con usuarios creados por el desarrollador, coherente con la
  [alta de nutricionistas por seed](decisiones/alta-de-nutricionistas-por-seed.md).

## 5. Ajustes al código que necesita staging

Las cards no los detallan, pero sin ellos el despliegue no funciona o sale más caro.
Corresponden a la card de staging, no a tareas del Incremento 2.

| Repo | Ajuste | Motivo |
|---|---|---|
| `nutria-frontend` | Imagen de producción: `npm run build` y archivos estáticos servidos por un servidor liviano (por ejemplo nginx), con reescritura de las rutas a `index.html` para React Router. | El `Dockerfile` actual corre `vite dev`. |
| `nutria-frontend` | `VITE_API_URL` como build arg en el CD. | Vite fija las variables `VITE_*` en el build, no al arrancar el contenedor. |
| `nutria-frontend` | Aviso de "entorno de prueba", activado por una variable de build. | Regla del entorno (card). |
| `nutria-backend` | `Dockerfile` de producción con `npm ci --omit=dev` (multi-stage). | Imagen más chica: menos almacenamiento y arranque en frío más corto. |
| `nutria-backend` | Poder desactivar el `prisma migrate deploy` del `CMD` en staging, porque las migraciones ya las corre el CD antes de activar la versión. | Con escala a cero, cada arranque en frío volvería a migrar. |
| `nutria-backend` | `directUrl` en el `datasource` de Prisma: conexión pooled en ejecución (pooler de transacciones, `?pgbouncer=true&connection_limit=1`) y pooler de sesión para `migrate` y el seed. | La conexión directa de Supabase puede ser solo IPv6, y ni los runners de Actions ni Cloud Run salen por IPv6. Las migraciones no funcionan a través del pooler de transacciones. |
| `nutria-backend` | `src/clients/ia.client.js`: pedir un identity token (audiencia = URL del servicio de IA) y mandarlo como `Authorization: Bearer`. Solo en Cloud Run: en compose se sigue llamando sin token. | Sin este cambio, el servicio de IA privado responde 403. |
| `nutria-backend` | `CORS_ORIGIN` con la URL del frontend en Cloud Run y `NODE_ENV=production` (oculta Swagger). `IA_TIMEOUT_MS` más alto en staging. | El arranque en frío del servicio de IA (carga del modelo ONNX) puede pasar los 5 s actuales. |
| `nutria-backend` | Una forma de correr el seed contra staging: un job en el CD o un script con la URL del pooler. | Paso 3 de la operación bajo demanda. |
| `nutria-ia` | Imagen sin dependencias de desarrollo y con el modelo de embeddings incluido, sin descargarlo al arrancar. Respetar el límite de 0,5 GB que verifica el CI. | Arranque en frío predecible. |

## 6. Alternativas descartadas para staging

| Alternativa | Por qué no |
|---|---|
| OCI Always Free (VM con el compose actual) | Costo cero real y sin arranques en frío, pero hay que administrar la VM a mano (SO, TLS, actualizaciones) y la capacidad gratuita no está garantizada. Queda como plan B si Cloud Run o Supabase fallan cerca de una presentación, porque no requiere los ajustes de la sección 5. |
| Frontend estático en Firebase Hosting o Cloudflare Pages | Ahorraría un servicio de Cloud Run, pero la card unifica los tres servicios en Cloud Run. Es una optimización posible para producción. |
| Keep-alive de Supabase (cron cada 3 días) | Innecesario con uso bajo demanda: se reanuda a mano antes de cada sesión. |
| Render / Railway / Fly.io | Sus capas gratuitas cambian seguido, y no tienen un servicio privado con autenticación entre servicios tan directo como Cloud Run. |
| Infraestructura propia | Expone una IP doméstica y no tiene garantía de disponibilidad. |

## 7. Inconsistencias a alinear

No se tocaron. Quedan anotadas para resolverlas cuando corresponda:

- **Proveedor de IA.** `AGENTS.md` da Gemini por cerrado y el Informe de Avances 2 lo
  tiene en evaluación (DA-12).
- **DA-14** sigue "en evaluación" en el Informe de Avances 2. Se actualiza en el próximo
  informe (staging resuelto, producción pendiente). Cuando se implemente, registrarla en
  `docs/decisiones/` a partir de la plantilla.
