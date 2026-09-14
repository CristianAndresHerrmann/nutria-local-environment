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
