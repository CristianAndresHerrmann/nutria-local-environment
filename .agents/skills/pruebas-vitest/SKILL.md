---
name: pruebas-vitest
description: Cómo organizar y escribir tests del backend de NutrIA con Vitest, y qué casos son obligatorios. Usar al agregar o modificar lógica de negocio en el backend.
---

# Pruebas — backend (Vitest)

Este skill opera dentro de `nutria-backend/`. Vitest es el runner asumido (si la
card T8 terminó definiendo Jest en su lugar, ajustar este skill en una línea antes
de usarlo).

Alcance del proyecto: **pruebas unitarias son obligatorias**; integración y
E2E/UI se agregan de forma selectiva, no como default de cada tarea.

## Cómo correrlos

Siempre dentro del contenedor, desde la raíz `nutria/` (no desde `nutria-backend/`):

```bash
npm run test:backend             # suite completa, una vez
npm run test:backend:coverage    # + cobertura: tabla en terminal y reporte en nutria-backend/coverage/ (index.html, lcov.info)
```

Eso hace `docker compose run --build --rm --no-deps backend npm run test`:
`--build` rebuildea la imagen (con cache de capas) antes de correr, así que un
test nuevo o un cambio en el código se ve sin ningún paso manual — el rebuild
cacheado tarda segundos porque `npm install` no se repite salvo que
`package.json`/`package-lock.json` hayan cambiado. `--no-deps` porque la suite
unitaria no toca Postgres (repositorios fake). No hay ningún archivo de compose
extra ni mounts: es el compose normal de `nutria-backend/` con dos flags.

Correr `npm run test` directo en `nutria-backend/` sirve para iterar rápido (o
`npm run test:watch`), pero la verificación que cuenta para cerrar una tarea es la
del contenedor.

Los archivos de test son CommonJS como el resto del repo y usan `describe`/`it`/
`expect`/`vi` como globales (`globals: true` en `vitest.config.js`): no hacer
`require('vitest')`, falla.

## Organización

- Un archivo de test por módulo de servicio/repositorio, junto al código que
  prueba (o en la carpeta de tests que ya exista en el repo — no crear una
  estructura paralela nueva).
- **Repositorios fake** en vez de pegarle a la base real en tests unitarios: el
  servicio se testea contra una implementación en memoria del repositorio, no
  contra Prisma + Postgres. Los tests que sí necesitan la base van aparte
  (integración), no se mezclan en la misma suite que corre en cada commit.

## Casos obligatorios (cuando el endpoint/servicio los involucra)

- **Aislamiento por nutricionista:** un nutricionista no puede leer ni modificar
  datos de pacientes de otro nutricionista.
- **Auth:** rutas protegidas rechazan requests sin token válido o con token de
  otro usuario.
- **Guardado parcial:** un plan o formulario que se guarda incompleto (borrador)
  se recupera tal cual se dejó, sin perder datos ni completarlos con valores
  inventados.
- **Menor sin responsable:** alta o edición de un paciente menor de edad sin
  responsable asociado se rechaza (o exige el dato), según la regla vigente —
  no inventar el comportamiento si no está definido, preguntar.
- **Login:** credenciales válidas autentican, inválidas no, y no hay fuga de
  información sobre si el error fue el usuario o la contraseña.

## Al terminar una tarea

El PR de la tarea no se considera completo sin al menos los tests obligatorios
que apliquen a lo que tocó. Si la tarea no toca ninguno de esos casos, el PR debe
igual incluir un test que cubra el camino principal nuevo.
