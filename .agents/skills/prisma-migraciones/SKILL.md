---
name: prisma-migraciones
description: Cómo hacer un cambio de modelo seguro en nutria-backend con Prisma. Usar al modificar schema.prisma o al necesitar una migración.
---

# Migraciones con Prisma — backend

Este skill opera dentro de `nutria-backend/`. Prisma es la herramienta de acceso a
datos y migraciones del proyecto (decisión cerrada).

## Flujo para cualquier cambio de modelo

1. Editar `schema.prisma`.
2. Generar migración **nombrada** y descriptiva:
   `npx prisma migrate dev --name <verbo_entidad_cambio>` (ejemplo:
   `agrega_campo_telefono_paciente`). Nunca dejar el nombre autogenerado
   genérico si Prisma lo pide.
3. Revisar el SQL generado en `prisma/migrations/` antes de commitear — Prisma no
   siempre infiere la migración más segura (por ejemplo, un campo NOT NULL nuevo
   sin default en una tabla con datos).
4. Actualizar el seed si el cambio de modelo lo requiere.
5. **Verificación desde cero:** correr `prisma migrate reset` (o levantar la base
   desde cero con docker-compose) y confirmar que migración + seed dejan la base
   en un estado consistente, no solo que el `migrate dev` incremental funcionó en
   tu base local ya existente.
6. Commitear el `schema.prisma` y la carpeta de migración juntos.

## Prohibido

- `prisma db push` fuera de un prototipado descartable que no se commitea. Todo
  cambio que llega a un PR va por migración nombrada.
- Texto libre en columnas donde el modelo de datos del IA1 define una entidad
  propia (por ejemplo, no guardar un plan alimentario como JSON de texto libre si
  el modelo ya tiene `OPCION_ITEM`/`ALIMENTO` para representarlo).

## Entidades que sostienen la validación determinística

`ALIMENTO`, `GRUPO_ALIMENTO`, `OPCION_ITEM`, `REGLA_CLINICA`, `HALLAZGO` — estas
cinco no se simplifican ni se aplanan en campos de texto; son la base de las
reglas clínicas deterministicas del backend.

## Convención de nombres

- Migraciones: `snake_case`, en español, verbo + qué cambia (ver ejemplo arriba).
- Si el cambio corresponde a una tarea con impacto de modelo relevante, anotarlo
  en `docs/decisiones/` (ver plantilla en la raíz de `nutria/`).
