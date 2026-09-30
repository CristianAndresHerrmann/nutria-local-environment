# Índice de decisiones técnicas

Una decisión por archivo, a partir de [`_plantilla.md`](_plantilla.md).

## Infraestructura y entorno

| Decisión | Tarea |
|---|---|
| [Docker Compose en el backend, referenciando los repos hermanos](compose-en-backend-con-repos-hermanos.md) | T3 |
| [Imagen `pgvector/pgvector:pg16` desde el inicio](imagen-pgvector-desde-el-inicio.md) | T3 |
| [Versiones fijadas de Node y Python](versiones-de-node-y-python.md) | T12 |

## Datos

| Decisión | Tarea |
|---|---|
| [Prisma para acceso a datos y migraciones](prisma-acceso-a-datos-y-migraciones.md) | T4 |
| [Ajustes al modelo de datos del Anexo A](ajustes-al-modelo-del-anexo-a.md) | T4 |

## Seguridad y acceso

| Decisión | Tarea |
|---|---|
| [Autenticación con JWT + bcrypt y aislamiento por nutricionista](autenticacion-jwt-y-aislamiento-por-nutricionista.md) | T5 |
| [Alta de nutricionistas solo por seed](alta-de-nutricionistas-por-seed.md) | T5 |

## API

| Decisión | Tarea |
|---|---|
| [Rutas de la API en inglés](rutas-de-api-en-ingles.md) | T6 |
| [Formato de error con detalle por campo](formato-de-error-por-campo.md) — *reemplazada* | T6 |
| [Formato de error uniforme `{ error: { code, message, details? } }`](formato-de-error-uniforme.md) | T7 |
| [Documentación de API con Swagger](documentacion-de-api-con-swagger.md) | Soporte |

## Frontend

| Decisión | Tarea |
|---|---|
| [Tailwind CSS v4 como framework de estilos](framework-de-estilos-tailwind.md) | T13 |

## Proceso de trabajo

| Decisión | Tarea |
|---|---|
| [Flujo de ramas y convención de commits](flujo-de-ramas.md) | T10 |
| [Asistentes de IA con contexto versionado](asistentes-de-ia-con-contexto-versionado.md) | T11 |
