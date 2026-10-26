# Índice de decisiones técnicas

Una decisión por archivo, a partir de [`_plantilla.md`](_plantilla.md).

El resumen con numeración DA para los informes de avance está en
[`registro-de-decisiones-incremento-2.md`](../registro-de-decisiones-incremento-2.md)
(Incremento 2, DA-15 a DA-24) y
[`registro-de-decisiones-incremento-3.md`](../registro-de-decisiones-incremento-3.md)
(Incremento 3, desde DA-25).

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
| [Ajustes al modelo de datos del Anexo A — Incremento 2](ajustes-al-modelo-del-anexo-a-incremento-2.md) | T14 |
| [Ajustes al modelo de datos del Anexo A — Incremento 3](ajustes-al-modelo-del-anexo-a-incremento-3.md) | T32 |
| [Catálogos comunes a todos los nutricionistas; recetas propias](catalogos-comunes-entre-nutricionistas.md) | T14 |
| [Catálogos curados por un administrador; alimentos privados por nutricionista](abm-de-catalogos-comunes.md) | T15 |
| [Nombres de catálogo únicos sin distinguir mayúsculas, con `citext`](nombres-de-catalogo-sin-distinguir-mayusculas.md) | T15 |
| [Recomendaciones clínicas comunes de solo lectura, y privadas por nutricionista](recomendaciones-privadas-por-nutricionista.md) | T22 |
| [Condiciones repetidas del paciente controladas en el servicio](condiciones-repetidas-controladas-en-el-servicio.md) | T18 |
| [La última actualización del paciente incluye sus consultas](ultima-actualizacion-del-paciente-incluye-consultas.md) | T19 |
| [IMC calculado al leer, con la talla más reciente a la fecha del peso](imc-calculado-con-la-talla-mas-reciente.md) | T20 |

## Seguridad y acceso

| Decisión | Tarea |
|---|---|
| [Autenticación con JWT + bcrypt y aislamiento por nutricionista](autenticacion-jwt-y-aislamiento-por-nutricionista.md) | T5 |
| [Alta de nutricionistas solo por seed](alta-de-nutricionistas-por-seed.md) — *modificada* | T5 |
| [Rol administrador y gestión de cuentas desde la aplicación](administrador-y-gestion-de-cuentas.md) | T31 |

## API

| Decisión | Tarea |
|---|---|
| [Rutas de la API en inglés](rutas-de-api-en-ingles.md) | T6 |
| [Formato de error con detalle por campo](formato-de-error-por-campo.md) — *reemplazada* | T6 |
| [Formato de error uniforme `{ error: { code, message, details? } }`](formato-de-error-uniforme.md) | T7 |
| [Documentación de API con Swagger](documentacion-de-api-con-swagger.md) | Soporte |
| [Aviso de datos clínicos faltantes calculado en el backend](aviso-de-datos-clinicos-faltantes.md) | T16 |
| [Contrato del servicio de IA para generar el plan, con proveedor simulado](contrato-del-servicio-de-ia.md) | T33 |

## Frontend

| Decisión | Tarea |
|---|---|
| [Tailwind CSS v4 como framework de estilos](framework-de-estilos-tailwind.md) | T13 |

## Pruebas

| Decisión | Tarea |
|---|---|
| [Estrategia de pruebas: un tipo de prueba por objetivo](estrategia-de-pruebas.md) | T23 |

## Proceso de trabajo

| Decisión | Tarea |
|---|---|
| [Flujo de ramas y convención de commits](flujo-de-ramas.md) | T10 |
| [Asistentes de IA con contexto versionado](asistentes-de-ia-con-contexto-versionado.md) | T11 |
