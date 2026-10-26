# Estrategia de despliegue a costo cero

**Fecha:** 25/10/2026 (reemplaza la versión del 21/10/2026, que elegía Cloud Run +
Supabase).
**Estado:** staging implementado en una VM de OCI (card "Desplegar staging", T-).
Producción queda sin definir.
**Fuente de la decisión:** card de Trello de staging. Si algo no coincide, manda la card.

## 1. Cuándo aplica

| Etapa | Entorno |
|---|---|
| Incremento 2 (en curso) | Local, con `npm run up` (docker-compose). |
| Incremento 3 | Local hasta terminarlo. Todavía no tiene tareas definidas. |
| Staging | VM de OCI con el mismo compose, para mostrar avances o para que la nutricionista pruebe algo. Se despliega con cada push a `staging`. |
| Producción | Se define cuando el sistema esté implementado. Fuera del alcance actual. |

docker-compose local sigue siendo el entorno de respaldo que siempre funciona (riesgo 13
del Plan de Proyecto). Staging usa ese mismo compose, así que lo que anda en local anda
en staging.

## 2. Antecedentes

- **Plan de Proyecto:** despliegue en "servicios gratuitos o de bajo costo". El riesgo 13
  pide mantener el sistema portable con Docker y, como contingencia, presentar en local.
- **Informe de Avances 2:** DA-14 dejaba la plataforma "en evaluación" entre
  infraestructura propia, OCI, GCP y servicios administrados similares. Esta decisión
  resuelve DA-14 **para staging**, y eso se informa en el próximo informe de avance.
- **21/10/2026:** se había elegido Cloud Run + Supabase, con OCI como plan B.
- **25/10/2026:** se cambia a OCI. Cloud Run exige una cuenta de facturación y la
  alerta de presupuesto de GCP avisa pero no corta el gasto; en OCI el sistema corre
  con el compose que ya existe, sin los ajustes que pedía Cloud Run (identity token,
  pooler de Supabase, `directUrl`, migraciones fuera del arranque).

## 3. Staging en OCI

```
Navegador
   │ HTTPS (Let's Encrypt)
   ▼
VM OCI Always Free (A1, ARM, Ubuntu 24.04) — docker compose
   Traefik ──► frontend (nginx, build estático)     https://<APP_HOST>
           └─► backend (Express)                     https://api.<APP_HOST>
                  ├──► ia (FastAPI) — sin ruta pública, solo red interna
                  │        └──► Gemini API (sin datos identificatorios)
                  └──► postgres (pgvector) — sin puerto publicado, volumen en la VM
```

| Componente | Plataforma y configuración |
|---|---|
| Cómputo | Una VM `VM.Standard.A1.Flex` (2 OCPU, 12 GB, disco de 100 GB), dentro del límite Always Free (4 OCPU, 24 GB, 200 GB). Región `us-ashburn-1`. |
| Red | VCN propia con subred pública. Security List: entrada solo en 22, 80 y 443. El firewall de Ubuntu abre los mismos puertos. |
| Servicios | `docker-compose.yml` de `nutria-backend` + `docker-compose.staging.yml`: HTTPS en Traefik, frontend con build estático, sin pgAdmin, sin puerto de Postgres, IA sin ruta pública, `restart: unless-stopped`. |
| Base de datos | El Postgres del compose, con volumen en la VM. Solo datos ficticios del seed, sin backups. El corpus del RAG va en una base separada de la operativa (DA-13). |
| Dominio y TLS | Subdominio gratuito de DuckDNS; Traefik pide y renueva el certificado de Let's Encrypt. |
| Build y despliegue | `cd-staging.yml` en cada repo, por push a `staging` o a mano: corre el CI, entra por SSH con una clave restringida a `scripts/deploy-staging.sh` y rebuildea el servicio en la VM. Smoke test por HTTPS. |
| Variables y secretos | Los de la aplicación, en los `.env` de la VM (no se versionan). Los del CD, en el Environment `staging` de GitHub. En los README se documentan solo los nombres. |

El detalle operativo (variables, primer arranque, clave de despliegue, comandos) está en
el README de `nutria-backend`, sección "Staging".

### Costo esperado

Cero. La VM, el disco, la red y el tráfico están dentro de Always Free; DuckDNS, Let's
Encrypt y GitHub Actions (repos públicos) son gratuitos. **Hay que verificar los límites
vigentes de Always Free si se cambia la VM.** El costo del primer mes se registra como
parte del criterio de cierre de la card.

### Barreras contra gastos imprevistos

La cuenta es Pay As You Go: lo Always Free no se cobra, pero cualquier recurso que no lo
sea sí.

1. **Alerta de presupuesto** de USD 1 en OCI.
2. **Crear solo recursos marcados *Always Free-eligible*** (shape A1, disco dentro de los
   200 GB totales, sin Load Balancer ni bases administradas).
3. **Cuota de solicitudes** en la clave de Gemini, cuando el servicio de IA la use.

A cambio, en Pay As You Go Oracle no recupera la VM por inactividad, como sí lo hace con
las cuentas Free Tier.

## 4. Operación bajo demanda (antes de cada sesión)

1. Confirmar que la VM está *Running* en la consola de OCI.
2. Recargar el seed de pacientes ficticios.
3. Abrir el frontend y hacer login una vez.

Si hace falta desplegar la versión vigente de `staging` sin un push nuevo, se corre
`cd-staging.yml` a mano (`workflow_dispatch`).

Reglas del entorno:
- Solo datos ficticios. Avisar explícitamente a la nutricionista que no cargue pacientes
  reales.
- Aviso visible en la interfaz de que es un entorno de prueba.
- Acceso solo con usuarios creados por el desarrollador, coherente con la
  [alta de nutricionistas por seed](decisiones/alta-de-nutricionistas-por-seed.md). En
  staging la contraseña del seed sale de una variable, porque el repo es público.

## 5. Alternativas descartadas para staging

| Alternativa | Por qué no |
|---|---|
| Cloud Run + Supabase (decisión del 21/10) | Exige cuenta de facturación y la alerta de presupuesto no corta el gasto. Además pedía ajustes de código propios de la plataforma (identity token, pooler, `directUrl`, migraciones fuera del arranque) y tenía arranques en frío. |
| Frontend estático en Firebase Hosting o Cloudflare Pages | Separaría el despliegue en dos plataformas; con la VM no ahorra nada. Es una optimización posible para producción. |
| Render / Railway / Fly.io | Sus capas gratuitas cambian seguido. |
| Infraestructura propia | Expone una IP doméstica y no tiene garantía de disponibilidad. |

## 6. Inconsistencias a alinear

- **Proveedor de IA.** `AGENTS.md` da Gemini por cerrado y el Informe de Avances 2 lo
  tiene en evaluación (DA-12).
- **DA-14** sigue "en evaluación" en el Informe de Avances 2. Se actualiza en el próximo
  informe (staging en OCI, producción pendiente).
