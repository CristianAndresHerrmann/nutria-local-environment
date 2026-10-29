# Categoría complementaria para alimentos sin aporte a la composición

**Fecha:** 29/10/2026
**Tarea relacionada:** T28

## Contexto

RF-25 y el schema Zod exigen al menos una categoría por alimento, y el enum
`food_category_type` tenía seis: `protein`, `carbohydrate`, `fiber`, `vegetable`,
`healthy_fat` e `infusion`. El catálogo definitivo (T28) necesita alimentos que no
encajan en ninguna: azúcares, dulces, condimentos, salsas, grasas sólidas (manteca,
crema, margarina) y agua. Las recetas (T30) y las reglas de diabetes e hipertensión
(T37) necesitan poder nombrarlos.

La composición esperada de cada comida (DA-29) se calcula sobre esas categorías, así que
la categoría que se les dé afecta las advertencias del motor.

## Alternativas consideradas

- **Forzar la categoría más cercana** (azúcar como hidrato, manteca como grasa
  saludable). Sin cambio de esquema, pero el azúcar cumpliría "hidrato" en el desayuno y
  la manteca "grasa saludable": advertencias de composición incorrectas.
- **Dejarlos fuera del catálogo.** Sin cambio de esquema, pero las recetas no pueden
  listarlos como ingredientes, las reglas de diabetes e hipertensión pierden su objeto, y
  la IA los devolvería como ítems no reconocidos (DA-26).
- **Agregar `complementary` al enum.** Cuesta una migración y tocar el enum en tres
  repos, pero no genera falsos positivos y deja los alimentos disponibles.

## Decisión

Se agrega `complementary` a `food_category_type` (migración
`agrega_categoria_complementary`). Se usa sola, nunca combinada con otra categoría. La
composición esperada (DA-29) no la exige en ninguna comida, así que un alimento
complementario no cubre ni genera advertencias. Modifica RF-25 del Plan de Proyecto.

## Consecuencias

- El enum está replicado en tres lugares y se actualizó en los tres: `schema.prisma` del
  backend (con Swagger y la etiqueta de los hallazgos), `FoodCategoryType` de
  `nutria-ia` (parseo estricto: sin esto, un catálogo con un alimento complementario daría
  422 en cada generación) y `foodCategories.js` del frontend.
- Un ítem de grupo cuyos alimentos son todos complementarios (por ejemplo, "Sal,
  especias y hierbas") tiene la categoría `complementary` por intersección y no cubre
  ninguna categoría esperada (DA-30). Es el comportamiento buscado.
- El valor se agrega con `ALTER TYPE ... ADD VALUE`, que no se puede revertir sin
  recrear el tipo. Ningún alimento existente cambia de categoría.
- Los 38 alimentos con esta categoría y el resto del catálogo tienen origen **V**: la
  nutricionista los valida en la entrevista (ver `docs/catalogo-de-alimentos.md`).
