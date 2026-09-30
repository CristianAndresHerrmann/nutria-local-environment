# Imagen `pgvector/pgvector:pg16` para Postgres desde el inicio

**Tarea relacionada:** T3

## Contexto

El Incremento 1 solo necesita Postgres estándar. Más adelante el servicio de IA usa
recuperación aumentada (RAG) con embeddings de 384 dimensiones guardados en la misma
base, para lo que hace falta la extensión `pgvector`. El Postgres administrado
previsto para el despliegue (Supabase) también la ofrece.

## Alternativas consideradas

- **Imagen oficial `postgres:16` ahora y cambiar de imagen cuando llegue RAG.** La
  más liviana hoy. En contra: el cambio de imagen se hace a mitad del proyecto, con
  datos ya cargados, y cualquier incompatibilidad aparece tarde.
- **`pgvector/pgvector:pg16` desde el día uno.** Es Postgres 16 estándar con la
  extensión disponible. En contra: una imagen un poco más grande que no aporta nada
  al Incremento 1.

## Decisión

El servicio `postgres` del compose usa `pgvector/pgvector:pg16` desde el inicio. La
extensión queda **disponible pero no activada**: ninguna migración ejecuta `CREATE
EXTENSION vector` en este incremento.

## Consecuencias

- Integrar RAG no requiere cambiar de imagen ni migrar el volumen de datos. Activar
  la extensión será una migración nombrada más.
- El cliente `psql` del stack (`npm run db:shell`) usa la misma imagen, así que las
  versiones de cliente y servidor coinciden.
- Del lado de Prisma, la columna vectorial va a necesitar `Unsupported(...)` y SQL
  crudo (ver [prisma-acceso-a-datos-y-migraciones.md](prisma-acceso-a-datos-y-migraciones.md)).
