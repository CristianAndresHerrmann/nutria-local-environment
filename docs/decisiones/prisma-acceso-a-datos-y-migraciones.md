# Prisma para acceso a datos y migraciones

**Tarea relacionada:** T4

## Contexto

El backend (Node.js/Express) necesita leer y escribir en PostgreSQL y llevar el
esquema versionado. El modelo del Incremento 1 es chico (nutricionista y paciente),
pero crece en el Incremento 2 con las columnas clínicas y contextuales (RF-04 a
RF-14), así que los cambios de esquema tienen que ser reproducibles en cualquier
máquina y en el despliegue. Además, el modelo lógico es uno de los artefactos que se
presentan en el Informe de Avance 2: conviene que exista en un único lugar legible.

## Alternativas consideradas

- **SQL a mano con el driver `pg`.** Control total de las consultas. En contra: sin
  migraciones ni tipos; el esquema queda repartido entre scripts SQL y el código, y
  cada consulta repite el mapeo de columnas.
- **Query builder con migraciones (Knex).** Migraciones versionadas. En contra: el
  modelo se reconstruye leyendo la secuencia de migraciones; no hay un archivo que
  sea el modelo.
- **ORM con modelos en código (Sequelize, TypeORM).** Modelos y relaciones en JS. En
  contra: las migraciones se escriben aparte del modelo y hay que mantener las dos
  cosas sincronizadas a mano.
- **Prisma.** Un `schema.prisma` declarativo que es a la vez el modelo lógico y la
  fuente de las migraciones (`prisma migrate` las genera con nombre), un cliente
  generado a partir de ese esquema y Prisma Studio para inspeccionar datos.

## Decisión

Se usa **Prisma** como capa de acceso a datos y herramienta de migraciones del
backend. `prisma/schema.prisma` es el modelo lógico del proyecto. Todo cambio de
esquema entra como migración nombrada (`prisma migrate dev --name ...`); `prisma db
push` queda fuera salvo para prototipos descartables.

## Consecuencias

- El modelo lógico para el IA2 se toma directamente de `schema.prisma`.
- Convención de nombres: identificadores en camelCase en el esquema, mapeados con
  `@map` a columnas snake_case en Postgres (ver
  [ajustes-al-modelo-del-anexo-a.md](ajustes-al-modelo-del-anexo-a.md)).
- El contenedor del backend corre `prisma migrate deploy` antes de levantar el
  servidor: un `up` deja el esquema al día sin pasos manuales.
- Solo los repositorios (`src/repositories/`) hablan con Prisma. Eso permite testear
  servicios con repositorios en memoria y los repositorios con un cliente falso,
  sin base de datos.
- Prisma no tiene un tipo nativo para `vector` de pgvector. Cuando se integre RAG, la
  columna se declara como `Unsupported("vector(384)")` y las búsquedas por similitud
  se escriben con SQL crudo (`$queryRaw`).
- El procedimiento para cambiar el modelo queda en el skill `prisma-migraciones`.
