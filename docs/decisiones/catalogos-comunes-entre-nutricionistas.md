# Catálogos comunes a todos los nutricionistas; recetas propias de cada uno

**Fecha:** 14/10/2026
**Tarea relacionada:** T14

## Contexto

El Anexo A del IA1 vincula RECETA con NUTRICIONISTA ("mantiene"), pero no dice nada sobre la
pertenencia de los demás catálogos: grupos de alimentos, alimentos, patologías y
recomendaciones clínicas. Hay que decidirlo al crear las tablas, porque determina si llevan
`nutritionist_id` y si el aislamiento de RNF-05 aplica sobre ellos.

## Alternativas consideradas

- **Todos los catálogos por nutricionista.** Aislamiento total. En contra: cada nutricionista
  tendría que cargar su propia base de alimentos, las reglas clínicas del Incremento 3 deberían
  resolverse contra catálogos distintos por usuario, y RF-30 pide recomendaciones "reutilizables
  sin modificación entre pacientes", algo propio de un contenido estándar.
- **Alimentos, grupos, patologías y recomendaciones comunes; recetas propias.** Una única base
  controlada (RF-24: "base mínima controlada de carga manual"), contra la que el motor de reglas
  valida igual para todos. Las recetas reflejan el criterio y el estilo de cada profesional, y el
  Anexo A ya las asocia al nutricionista.

## Decisión

**Supuesto:** alimentos, grupos de alimentos, patologías y recomendaciones clínicas son catálogos
comunes, sin `nutritionist_id`. Las recetas tienen `nutritionist_id` y se filtran por el
nutricionista autenticado, igual que los pacientes (RNF-05). Hay que confirmarlo en la
entrevista de validación con nutricionistas.

## Consecuencias

- Hoy los catálogos comunes se cargan por seed. Si más adelante hay un ABM, editar un
  catálogo común afecta a todos los nutricionistas: ese ABM no puede quedar abierto a cualquier
  usuario sin una decisión explícita sobre quién lo administra. Resuelto en T15: el catálogo lo cura
  un administrador, los nutricionistas no lo editan, y lo que agrega cada uno es un alimento privado
  ([abm-de-catalogos-comunes.md](abm-de-catalogos-comunes.md)).
- Los repositorios de recetas aplican el mismo resguardo que `patient.repository` (cortar
  cualquier consulta sin `nutritionistId`). Los de catálogos comunes no lo necesitan.
- Si la validación con nutricionistas pide catálogos propios (por ejemplo, alimentos regionales
  de un consultorio), se agrega una columna `nutritionist_id` opcional por migración, donde
  `NULL` signifique "común", sin rehacer el modelo. Aplicado en T15 a los alimentos: `food.nutritionist_id`
  nullable, y desde entonces el repositorio de alimentos sí aplica el resguardo de
  `patient.repository`. Aplicado en T22 a las recomendaciones clínicas
  ([recomendaciones-privadas-por-nutricionista.md](recomendaciones-privadas-por-nutricionista.md)).
