# Nombres de catálogo únicos sin distinguir mayúsculas, con `citext`

**Fecha:** 15/10/2026
**Tarea relacionada:** T15

## Contexto

T15 pide que el nombre de un alimento sea único sin distinguir mayúsculas, y que un duplicado
responda `CONFLICT`. Si conviven "Leche" y "leche", resolver los ítems de un plan contra el
catálogo en el Incremento 3 se vuelve ambiguo.

T14 había dejado `name` con `@unique` en `food`, `food_group` y `pathology`, pero ese índice
distingue mayúsculas. Además, en T15 un alimento puede ser privado de un nutricionista
([abm-de-catalogos-comunes.md](abm-de-catalogos-comunes.md)), así que la unicidad de alimentos
pasa a depender de lo que ve cada uno.

## Alternativas consideradas

- **Chequeo solo en el servicio** (`findFirst` con `mode: 'insensitive'`). No toca la base, pero
  dos altas simultáneas pueden pasar el chequeo y crear el duplicado igual.
- **Índice único funcional sobre `lower(name)`** escrito a mano en la migración. Prisma no puede
  expresarlo en `schema.prisma`, así que lo trataría como drift o lo eliminaría en una migración
  posterior.
- **Columna `citext`** (`@db.Citext`). Prisma soporta el tipo nativo, y todo índice único sobre
  la columna compara sin distinguir mayúsculas. Solo hace falta crear la extensión `citext`, que
  Supabase también tiene disponible.

## Decisión

`name` pasa a `citext` en `food`, `food_group` y `pathology`, en la migración
`20261015180000_agrega_alimentos_privados_y_nombres_sin_mayusculas`.

- **Grupos y patologías:** el `@unique` global de `name` se mantiene.
- **Alimentos:** el nombre es único dentro de lo que ve cada nutricionista (los comunes más los
  suyos):
  - `@@unique([nutritionistId, name])` respalda en la base que un nutricionista no repita el
    nombre de un privado propio.
  - Postgres trata los `NULL` como distintos en un índice único, así que el cruce con los comunes
    lo chequea el servicio antes de escribir. Para que la base también lo cubriera haría falta
    `NULLS NOT DISTINCT` o índices parciales, que Prisma no expresa.
  - Dos nutricionistas sí pueden tener cada uno un privado con el mismo nombre.
- `CREATE EXTENSION IF NOT EXISTS citext;` se agrega a mano al principio del `migration.sql`.
  Sin el preview feature `postgresqlExtensions`, Prisma no crea extensiones ni las compara, y
  `migrate dev` responde "Already in sync".
- El servicio responde `409` con el campo en `details.fields.name`. La API normaliza los
  espacios de más (`"Pan  francés "` se guarda como `"Pan francés"`).

## Consecuencias

- **No ignora acentos:** "Cafe" y "Café" son nombres distintos. Para eso haría falta `unaccent`
  en un índice funcional, que tiene el mismo problema con Prisma descrito arriba.
- **Hay dos carreras que la base no detecta:** dos altas simultáneas del mismo común (las hacen
  el seed o el administrador), o un privado que se crea al mismo tiempo que un común con ese
  nombre. Se aceptan, porque las altas de comunes no son concurrentes.
- **El seed no puede hacer `upsert` por nombre** en los alimentos comunes: Prisma no acepta
  `null` en una clave compuesta. Busca entre los comunes y después actualiza o crea.
- **Reiniciar el backend si estaba levantado:** al aplicar esta migración, o al correr
  `npm run db:reset`, hay que reiniciarlo. Prisma cachea los tipos de la base y, hasta
  reiniciar, responde `500` ("cache lookup failed for type…"). `npm run up` no tiene el
  problema, porque migra antes de arrancar el server.
