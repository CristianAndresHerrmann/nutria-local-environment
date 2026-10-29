# Recetas básicas comunes

Origen, criterios y pendientes de las recetas básicas comunes que carga el seed (T30;
RF-27, RF-28; DA-34). Las recetas completas (ingredientes, cantidades, preparación y notas)
viven en `nutria-backend/scripts/seed-data/recipes.js` y no se repiten acá.

## Volumen y origen

| | |
|---|---|
| Desayuno y merienda | 7 (4 sin cocción) |
| Almuerzo y cena | 10 (2 sin cocción) |
| Origen | **V** (pendiente de validación) en todas |

Es una propuesta del desarrollador, sin revisión de una nutricionista. Las recetas
elaboradas o con un proceso detallado no van acá: cada nutricionista las carga como
privadas, o duplica una básica y la adapta.

## Criterios

- **Volumen.** T30 pide al menos 2 recetas por comida. Se amplía porque la generación
  recorta las candidatas por paciente: si el paciente no cocina, solo pasan las recetas sin
  cocción (DA-31), y además se aplican el filtro de RF-28 y las reglas por patología.
- **Comidas.** Las recetas de desayuno y merienda se ofrecen en las dos comidas, y lo mismo
  las de almuerzo y cena.
- **Ingredientes.** Solo alimentos comunes del catálogo (DA-23), con su nombre exacto y sin
  repetir dentro de una receta. La preparación no nombra alimentos que no figuren como
  ingredientes; el agua de cocción es la única excepción.
- **Alimentos evitados.** Ninguna receta lleva sal, azúcar, dulces, caldo en cubo, salsa de
  soja ni fiambres: con alguno, una regla de patología podría excluir la receta completa.
  Se condimentan con hierbas, especias, limón y aceite.
- **Composición (DA-29).** Las de desayuno y merienda incluyen proteína e hidrato; las de
  almuerzo y cena, verdura, proteína e hidrato, según las categorías de sus ingredientes.
- **Cantidades.** En medidas caseras culinarias, no son una prescripción. El rendimiento
  (porciones) va al inicio de la nota final, porque la receta no tiene un campo propio.

## Listado

| # | Receta | Comidas | Cocción | Tiempo | Rinde |
|---|---|---|---|---|---|
| 1 | Pan integral con queso untable y tomate | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 2 | Yogur con granola y frutillas | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 3 | Licuado de banana con leche y avena | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 4 | Yogur con banana y semillas de lino | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 5 | Tostadas de arroz con palta y huevo duro | Desayuno y merienda | Anafe | 15 min | 1 |
| 6 | Avena cocida con leche y pera | Desayuno y merienda | Anafe | 10 min | 1 |
| 7 | Panqueques de avena y banana | Desayuno y merienda | Anafe | 15 min | 1 |
| 8 | Pollo al horno con papas y zanahorias | Almuerzo y cena | Horno | 60 min | 1 |
| 9 | Tarta de acelga y huevo | Almuerzo y cena | Horno | 50 min | 4 |
| 10 | Fideos con salsa de tomate y carne picada | Almuerzo y cena | Anafe | 30 min | 1 |
| 11 | Arroz con pollo y verduras | Almuerzo y cena | Anafe | 35 min | 1 |
| 12 | Milanesa de carne al horno con ensalada | Almuerzo y cena | Horno | 35 min | 1 |
| 13 | Merluza al horno con papa y zapallo | Almuerzo y cena | Horno | 40 min | 1 |
| 14 | Guiso de lentejas con verduras | Almuerzo y cena | Anafe | 50 min | 2 |
| 15 | Tortilla de papa con tomate | Almuerzo y cena | Anafe | 35 min | 2 |
| 16 | Ensalada de garbanzos, choclo y tomate | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 17 | Wrap de pan árabe con atún y vegetales | Almuerzo y cena | Sin cocción | 10 min | 1 |

## Verificación

`nutria-backend/scripts/seed-data/recipes.test.js` controla los datos sin base: nombres de
receta únicos, ingredientes del catálogo común sin repetir, sin alimentos evitados, al menos
2 recetas y 2 sin cocción por tipo de comida, valores válidos y la composición esperada de
cada comida en que se ofrecen. Además, el seed las carga por el servicio de recetas, que
rechaza un ingrediente que no sea común.

## Pendientes para la entrevista de validación

- [ ] Revisar el listado: recetas que faltan o sobran, y preparaciones que en la consulta se
  hacen distinto.
- [ ] Confirmar las cantidades en medidas caseras, o definir si las recetas comunes van sin
  cantidades.
- [ ] Confirmar si la receta 4 completa un desayuno: el hidrato lo aporta la banana, no un
  cereal.
- [ ] Decidir si las recetas comunes deben evitar la sal y el azúcar, o si se cargan con
  ellas y el motor las excluye según la patología.
- [ ] Confirmar el supuesto de DA-15 modificado por DA-34: recetas básicas comunes más las
  propias de cada nutricionista.
