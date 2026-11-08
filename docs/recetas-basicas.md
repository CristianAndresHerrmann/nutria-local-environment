# Recetas básicas comunes

Origen, criterios y pendientes de las recetas básicas comunes que carga el seed (T30;
RF-27, RF-28; DA-34). Las recetas completas (ingredientes, cantidades, preparación y notas)
viven en `nutria-backend/scripts/seed-data/recipes.js` y no se repiten acá.

## Volumen y origen

| | |
|---|---|
| Desayuno y merienda | 27 (18 sin cocción) |
| Almuerzo y cena | 50 (17 sin cocción) |
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
- **Ampliación del 07/11/2026.** Las recetas 18 a 77 salen de un set de 158 recetas ficticias
  armado para probar la generación, con los mismos criterios. Van como comunes las más simples.
  Otras 40, las elaboradas o con ingredientes menos habituales, se cargan como privadas de
  `nutricionista@nutria.com` (`seed-data/private-recipes.js`), junto con 5 de control que llevan
  a propósito un alimento alcanzado por una regla. Las 58 restantes no se cargan.

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
| 18 | Tostadas de arroz con ricota y frutillas | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 19 | Yogur griego con copos de maíz y arándanos | Desayuno y merienda | Sin cocción | 3 min | 1 |
| 20 | Licuado de frutilla y banana con leche | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 21 | Pan lactal integral con queso fresco y palta | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 22 | Galletitas sin TACC con queso untable y kiwi | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 23 | Avena remojada con yogur y durazno | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 24 | Avena sin TACC remojada con bebida de soja y manzana | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 25 | Mate cocido con leche y pan integral con ricota | Desayuno y merienda | Anafe | 5 min | 1 |
| 26 | Café con leche y pan francés con queso port salut | Desayuno y merienda | Microondas | 5 min | 1 |
| 27 | Té con leche y galletitas de salvado con queso untable | Desayuno y merienda | Microondas | 5 min | 1 |
| 28 | Yogur bebible con granola y banana | Desayuno y merienda | Sin cocción | 3 min | 1 |
| 29 | Pan de salvado con huevo revuelto y tomate | Desayuno y merienda | Anafe | 10 min | 1 |
| 30 | Omelette de espinaca y queso con pan integral | Desayuno y merienda | Anafe | 10 min | 1 |
| 31 | Pan sin TACC con huevo revuelto y palta | Desayuno y merienda | Anafe | 10 min | 1 |
| 32 | Yogur descremado con copos de maíz y mango | Desayuno y merienda | Sin cocción | 3 min | 1 |
| 33 | Sándwich de pan árabe con queso de máquina y tomate | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 34 | Licuado de durazno con yogur y avena | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 35 | Licuado de mango y banana con leche deslactosada | Desayuno y merienda | Sin cocción | 5 min | 1 |
| 36 | Leche con copos de maíz y banana | Desayuno y merienda | Sin cocción | 3 min | 1 |
| 37 | Pan francés con queso fresco y mandarina | Desayuno y merienda | Sin cocción | 3 min | 1 |
| 38 | Ensalada de lentejas, choclo y morrón | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 39 | Ensalada de garbanzos y pepino con pan árabe | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 40 | Ensalada de atún, choclo y lechuga con tostadas de arroz | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 41 | Sándwich integral de queso fresco, tomate y rúcula | Almuerzo y cena | Sin cocción | 5 min | 1 |
| 42 | Wrap de pan árabe con hummus y vegetales | Almuerzo y cena | Sin cocción | 15 min | 2 porciones de hummus |
| 43 | Ensalada de porotos negros, choclo y palta | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 44 | Ensalada caprese con pan integral | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 45 | Ensalada de ricota, rúcula y nueces con galletitas integrales | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 46 | Sándwich de pan sin TACC con atún y zanahoria | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 47 | Ensalada de porotos alubia con atún y tomate | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 48 | Tostadas de arroz con palta, tomate y queso fresco | Almuerzo y cena | Sin cocción | 5 min | 1 |
| 49 | Sándwich de pan lactal integral con ricota, espinaca y tomate | Almuerzo y cena | Sin cocción | 5 min | 1 |
| 50 | Ensalada de porotos colorados y tomate con pan árabe | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 51 | Ensalada de atún, espinaca y manzana con galletitas de salvado | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 52 | Ensalada de queso port salut, lechuga y choclo con pan francés | Almuerzo y cena | Sin cocción | 10 min | 1 |
| 53 | Pollo a la plancha con puré de papa y zapallo | Almuerzo y cena | Anafe | 30 min | 1 |
| 54 | Bife magro con ensalada y papas al horno | Almuerzo y cena | Horno | 40 min | 1 |
| 55 | Merluza a la plancha con arroz integral y brócoli | Almuerzo y cena | Anafe | 35 min | 1 |
| 56 | Hamburguesas caseras de carne con pan y ensalada | Almuerzo y cena | Anafe | 25 min | 2 hamburguesas |
| 57 | Hamburguesas de lentejas con pan integral y ensalada | Almuerzo y cena | Anafe | 40 min | 4 hamburguesas |
| 58 | Medallones de garbanzo con puré de batata | Almuerzo y cena | Horno | 40 min | 4 medallones |
| 59 | Tarta de zapallitos y queso | Almuerzo y cena | Horno | 50 min | 4 |
| 60 | Tarta de brócoli y ricota | Almuerzo y cena | Horno | 50 min | 4 |
| 61 | Fideos integrales con brócoli, ajo y queso rallado | Almuerzo y cena | Anafe | 20 min | 1 |
| 62 | Fideos de arroz salteados con pollo y vegetales | Almuerzo y cena | Anafe | 25 min | 1 |
| 63 | Fideos sin TACC con salsa de tomate y atún | Almuerzo y cena | Anafe | 20 min | 1 |
| 64 | Arroz cremoso con champiñones y pollo | Almuerzo y cena | Anafe | 35 min | 1 |
| 65 | Polenta gratinada con espinaca y queso | Almuerzo y cena | Horno | 30 min | 2 |
| 66 | Quinoa con verduras salteadas y huevo | Almuerzo y cena | Anafe | 25 min | 1 |
| 67 | Ensalada tibia de quinoa, garbanzos y calabaza asada | Almuerzo y cena | Horno | 40 min | 2 |
| 68 | Arroz integral con lentejas y cebolla caramelizada | Almuerzo y cena | Anafe | 45 min | 2 |
| 69 | Milanesas de pollo al horno con puré mixto | Almuerzo y cena | Horno | 45 min | 2 milanesas |
| 70 | Milanesas de berenjena al horno con queso | Almuerzo y cena | Horno | 40 min | 2 |
| 71 | Revuelto de arvejas, papa y huevo | Almuerzo y cena | Anafe | 25 min | 1 |
| 72 | Tortilla de verduras al horno | Almuerzo y cena | Horno | 35 min | 2 |
| 73 | Frittata de brócoli y queso con batata | Almuerzo y cena | Horno | 40 min | 2 |
| 74 | Wok de carne con brócoli y arroz | Almuerzo y cena | Anafe | 25 min | 1 |
| 75 | Pollo al limón con arroz integral y chauchas | Almuerzo y cena | Anafe | 40 min | 1 |
| 76 | Croquetas de atún y papa al horno con ensalada | Almuerzo y cena | Horno | 40 min | 2 |
| 77 | Ensalada de pollo, arroz integral y palta | Almuerzo y cena | Anafe | 35 min | 1 |

## Verificación

`nutria-backend/scripts/seed-data/recipes.test.js` controla los datos sin base: nombres de
receta únicos, ingredientes del catálogo común sin repetir, sin alimentos evitados, al menos
2 recetas y 2 sin cocción por tipo de comida, valores válidos y la composición esperada de
cada comida en que se ofrecen. Además, el seed las carga por el servicio de recetas, que
rechaza un ingrediente que no sea común.

`clinical-cases.rules.test.js`, en la misma carpeta, corre el filtro real de la generación con
estas recetas para los casos C1 a C7: ninguna candidata alcanzada por una condición o una regla
prohibitiva, y al menos 5 candidatas por comida en cada caso.

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
- [ ] Revisar recetas que ninguna regla alcanza hoy y que el sistema ofrece: con conservas, quesos
  o aceitunas en hipertensión, con arroz con leche o fruta desecada en diabetes, y con yogur o
  queso cuando la intolerancia es a `Leche`, porque la condición alcanza solo a ese alimento. Son
  preguntas para la entrevista, no reglas: el sistema no las restringe.
