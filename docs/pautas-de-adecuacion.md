# Pautas de adecuación nutricional

Qué conviene ofrecer en un plan, según la patología del paciente, su objetivo y las pautas
para la población general. Cada pauta tiene su cita con página y se expresa como un
indicador que se puede medir sobre las opciones del plan. Es la referencia de evaluación
de T58 e insumo de T45, T61 y T65 (Plan 5.6; riesgos 15 y 18; DA-13, DA-29). Fundamento en
[DA-41](decisiones/pautas-de-adecuacion-v1.md).

## Propósito y alcance

Las [reglas clínicas](reglas-clinicas.md) dicen qué evitar. Las pautas dicen qué conviene
incluir. Son cosas distintas y viven en lugares distintos:

| | Reglas (R-01 a R-06) | Pautas (este documento) |
|---|---|---|
| Qué expresan | Lo que no puede o no debería estar en el plan | Lo que conviene que esté |
| Quién las aplica | El motor determinístico, en cada guardado | La evaluación de T58, sobre planes ya generados |
| Efecto en el editor | Hallazgo `blocking` o `warning` | Ninguno: no bloquean ni advierten |
| Cómo llegan a la IA | Por el catálogo permitido (DA-31, DA-40) | Por el RAG, desde el corpus (T45, DA-13). Este documento no se copia al prompt |

- **La seguridad sigue en el motor (riesgo 15).** Una pauta nunca habilita algo que una
  regla restringe, y un alimento `prohibited` nunca cuenta como cumplimiento.
- **Las pautas no se duplican.** Este documento es la vara para medir. El RAG es el
  mecanismo que acerca la bibliografía a la IA, y recupera las fuentes, no esta tabla.
  Si el RAG mejora el plan, tiene que notarse en estos indicadores.
- **Sin pautas inventadas (riesgo 18).** Cada pauta sale de una fuente del corpus, con la
  cita literal verificada contra el texto de su página. Lo que no tiene fuente queda **V**
  y sin completar.

## Origen

Mismos códigos que en [reglas-clinicas.md](reglas-clinicas.md#origen-de-una-regla). Acá se
usan dos:

- **S**: la pauta, con su fuente (obra y página). Toda pauta de este documento es S.
- **V**: los [criterios de medición](#criterios-de-medición), que traducen la pauta al
  catálogo, y los objetivos sin guía. Son una interpretación del desarrollador.

El origen lo registra una persona en DA-41. Cuando la entrevista (T65) confirme una pauta
o un criterio, pasa a **E**.

Cada pauta se identifica como `PA-<clave>-NN`. La clave es `GEN` (población general),
`DM2`, `HTA` o `CEL` (patologías), u `OBE` (objetivo bajar de peso).

## Fuentes

Las citas son de la página impresa. En todas las fuentes coincide con la página del PDF
(`pagina_pdf` en el corpus), salvo en MAN, donde el desfase va de +1 a +6. Ahí se indican
las dos, porque los fragmentos del RAG se referencian por `pagina_pdf`.

| Clave | Obra | Corpus | Condiciones de uso |
|---|---|---|---|
| GAPA | Ministerio de Salud de la Nación. *Guías alimentarias para la población argentina*. 2016. | ID 01, `02_fuentes_por_revisar` | Por comprobar |
| MAN | Ministerio de Salud de la Nación. *Manual para la aplicación de las Guías Alimentarias para la Población Argentina*. 2018. | ID 02, `02_fuentes_por_revisar` | Por comprobar |
| DM2 | Ministerio de Salud de la Nación. *Guía de Práctica Clínica Nacional sobre Prevención, Diagnóstico y Tratamiento de la Diabetes Mellitus Tipo 2*. 2019. | ID 08, `01_corpus_rag_inicial` | Acceso y uso libre, citando adecuadamente |
| HTA | Ministerio de Salud de la Nación. *Guía de Práctica Clínica Nacional. Prevención, Diagnóstico y Tratamiento de la Hipertensión Arterial — versión breve*. 2026. | ID 09, `02_fuentes_por_revisar` | Por comprobar |
| CEL | Ministerio de Salud de la Nación. *Guía de práctica clínica sobre diagnóstico y tratamiento de la enfermedad celíaca en el primer nivel de atención*. 2016. | ID 13, `02_fuentes_por_revisar` | Por comprobar |
| ALG | ANMAT / INAL. *Guía de recomendaciones para un menú libre de gluten seguro*, 2.ª ed. 2023. | ID 14, `02_fuentes_por_revisar` | Por comprobar |
| OBE | Ministerio de Salud de la Nación. *Guía de Práctica Clínica Nacional sobre Abordaje Integral de la Obesidad en personas adultas — versión breve para equipos de salud*. 2023. | ID 12, `01_corpus_rag_inicial` | Acceso y uso libre, citando adecuadamente (p. 5) |

- Las URL de cada obra están en el inventario del corpus
  (`datos-rag/corpus-v1/04_metadatos/bibliografia.md`).
- **OBE es la versión breve.** Trae las 15 recomendaciones de la guía 2023, que reemplaza a
  la de 2013. La versión extensa no está en el corpus, y su incorporación queda para T46.
  Verificada el 13/11/2026:
  - URL: `https://www.argentina.gob.ar/sites/default/files/bancos/2023-11/guia_practica_clinica_nacional_sobre_abordaje_integral_obesidad_personas_adultas_14112023.pdf`
    (responde, PDF de 174 páginas, generado el 13/11/2023).
  - Versión: es la guía completa, de la que se derivan las versiones abreviadas.
  - Condiciones de uso, p. 14: "Este y todos los materiales derivados de la guía son de
    acceso y uso libre, citándolos adecuadamente".
  - 117 de las 174 páginas tienen el texto con una codificación de fuente rota, y no se
    extrae. Para incorporarla, T46 necesita OCR.
- **En OBE, parte de las pp. 30, 31 y 33 es imagen, sin texto en el corpus**: los grupos de
  la GAPA (p. 30), el grupo de consumo opcional (p. 31) y el plato por comida (p. 33). Se
  leyeron a mano el 13/11/2026 y se citan como "p. N, imagen". El RAG no las recupera,
  porque sus fragmentos salen del texto extraído. La p. 29 es la gráfica de la GAPA, sin
  pautas propias.
- **Solo DM2 y OBE están en el corpus inicial** (`chunks.jsonl`). Las demás están en
  `02_fuentes_por_revisar`, y el RAG no las recupera hasta que T46 las habilite. Mientras
  tanto, una pauta de GAPA, MAN, HTA, CEL o ALG se puede medir, pero el RAG no la lleva a la
  IA.

## Criterios de medición

Cómo se traduce una pauta al plan. Todos son **V**.

### Unidades

Un plan tiene opciones por comida (desayuno, almuerzo, merienda y cena), y cada opción tiene
ítems: un alimento, un grupo, una receta o un ítem no reconocido. El plan no tiene cantidades,
días ni semanas. Por eso hay dos niveles:

| Nivel | Cuándo | Indicador |
|---|---|---|
| **Opción** | La fuente ata la pauta a una comida ("almuerzo y cena", "desayunos o meriendas") | Porcentaje de opciones de esa comida que cumplen |
| **Plan** | La fuente la plantea para el día ("a diario", "por día") | Cumple si al menos una opción del plan, de las comidas que nombre la fuente, la cumple |

El nivel plan es la lectura más débil de una pauta diaria: un plan con una sola opción de
cena con pescado cumple "aumentar el consumo de pescado". Es lo único medible sin cantidades
ni días, y queda como pregunta para T65.

### Qué aporta un ítem

| Criterio | Definición | Reutiliza |
|---|---|---|
| CM-01 Categorías | Igual que el motor: el alimento aporta su copia de `food_categories`; la receta, la unión de las de sus ingredientes; el grupo, la intersección de las de sus alimentos vigentes; el no reconocido, ninguna. | `itemCategories.js` (DA-16, DA-30) |
| CM-02 Pertenencia a un grupo | Un alimento pertenece a su grupo hoja y a los grupos que lo contienen. Una receta pertenece a los grupos de sus ingredientes. Un ítem de grupo pertenece a un grupo de la pauta si está comprendido en él. Un ítem de grupo más amplio que el de la pauta no cuenta (por ejemplo, "Carnes, pescados y huevos" no cumple una pauta de Pescados). | `scopeCoversFood` en `src/utils/recipeConditionFilter.js` (DA-23, DA-30) |
| CM-03 Restringidos | Un ítem que dispara un hallazgo `blocking` del motor no cuenta para ninguna pauta. Ante un conflicto entre fuentes, prevalece la de la patología: los cereales con gluten de la GAPA no cuentan en un celíaco. | DA-36, DA-40 |
| CM-10 Condimentos | Un alimento que se usa en cantidad de condimento no cuenta como porción de su grupo, aunque el catálogo le dé la categoría: Limón no cuenta para Frutas, Ajo no cuenta para Verduras, y Queso rallado no cuenta para Lácteos ni como proteína. Limón y Ajo sí cuentan para PA-HTA-02. El motor (R-03) no hace esta excepción. | DA-42 |

### Atributos que el catálogo no tiene

El catálogo no marca si un alimento es integral, descremado o magro. Se aproximan así
(correcciones de DA-42):

| Criterio | Definición |
|---|---|
| CM-04 Integral | Un alimento de Cereales y derivados con categoría `fiber`, salvo Cebada perlada (el perlado le quita el salvado), Granola (suele tener azúcar agregada), Salvado de avena y Salvado de trigo (son un agregado, no una porción de cereal). Hoy: Pan integral, Pan lactal integral, Pan de salvado, Harina integral, Fideos integrales, Arroz integral, Avena arrollada, Avena sin TACC, Trigo burgol, Quinoa, Amaranto, Galletitas integrales y Galletitas de salvado. |
| CM-05 Descremado o magro | Un lácteo con "descremada" o "descremado" en el nombre, más la Ricota, que es magra aunque sea entera: Leche descremada, Leche parcialmente descremada, Yogur descremado, Queso untable descremado, Ricota y Ricota descremada. Un ítem de grupo ("un lácteo") queda sin determinar: sale del denominador. |
| CM-06 Carnes rojas y cortes magros | Carnes rojas: el subgrupo Carnes (vacuna, cerdo y cordero). Cortes magros: Carne vacuna magra, Carne picada magra, Lomo de cerdo e Hígado vacuno. |

### A quién se aplica

| Criterio | Definición |
|---|---|
| CM-07 Patología | Una pauta de patología se aplica si el paciente tiene esa patología como condición. |
| CM-08 Objetivo | El objetivo del paciente (`patient.goal`) es texto libre. Una pauta de objetivo se aplica cuando el profesional o T58 identifican ese objetivo en el texto. No hay un mapeo automático. |
| CM-09 Población general | Las pautas `GEN` se aplican a todos los pacientes, con la salvedad de CM-03. |

## Pautas para la población general (GAPA y MAN)

Se aplican a todos los pacientes. Las fuentes de patología y de objetivo remiten a ellas
(HTA p. 14; OBE p. 28; CEL p. 43).

| ID | Pauta | Nivel y comidas | Indicador | Cita | Origen |
|---|---|---|---|---|---|
| PA-GEN-01 | Cuatro comidas al día | Plan | El plan tiene opciones de desayuno, almuerzo, merienda y cena. | GAPA p. 26: "Realizar 4 comidas al día (desayuno, almuerzo, merienda y cena)". También DM2 p. 251 y OBE p. 32. | S |
| PA-GEN-02 | Alimentos de todos los grupos | Plan | Cuántos de estos 8 aparecen en el plan: Verduras (sin Tubérculos y choclo), Frutas, Legumbres, Cereales y derivados, Lácteos, Huevos, carnes (Carnes, Aves, Pescados o Mariscos) y Aceites. Cumple con 8 de 8. | GAPA p. 26: "incluir verduras, frutas, legumbres, cereales, leche, yogur o queso, huevos, carnes y aceites." | S |
| PA-GEN-03 | Verduras en el almuerzo y en la cena | Opción; almuerzo y cena | La opción tiene un ítem de Verduras que no es de Tubérculos y choclo ni Ajo (CM-10). | GAPA p. 26: "Consumir al menos medio plato de verduras en el almuerzo, medio plato en la cena". DM2 p. 250, sobre papa, batata, choclo y mandioca: "No se incluyen en este grupos papa, batata, choclo y mandioca." También OBE p. 30, imagen: "No se incluyen en este grupo papa, batata, choclo y mandioca." | S |
| PA-GEN-04 | Frutas todos los días | Plan | Hay un ítem de Frutas frescas en el plan, sin contar Limón (CM-10). Además, se informa en cuántas comidas aparece. | GAPA p. 26: "2 o 3 frutas por día." | S |
| PA-GEN-05 | Lácteos todos los días, preferentemente descremados | Plan | Hay un ítem de Lácteos en el plan. Además, el porcentaje de ítems de Lácteos que son descremados (CM-05). | GAPA p. 27: "Consumir diariamente leche, yogur o queso, preferentemente descremados." | S |
| PA-GEN-06 | Pescado y huevo | Plan | Hay un ítem de Pescados en el plan. Hay un ítem de Huevos en el plan. Son dos indicadores. | GAPA p. 27: "aumentar el consumo de pescado e incluir huevo." | S |
| PA-GEN-07 | Legumbres y cereales integrales | Plan | Hay un ítem de Legumbres en el plan. Además, el porcentaje de ítems de Cereales y derivados que son integrales (CM-04). | GAPA p. 27: "Consumir legumbres, cereales preferentemente integrales, papa, batata, choclo o mandioca." | S |
| PA-GEN-08 | Aceite, frutas secas o semillas | Plan | Hay un ítem de Aceites o de Frutos secos y semillas en el plan. | GAPA p. 27: "Consumir aceite crudo como condimento, frutas secas o semillas." | S |
| PA-GEN-09 | Desayunos y meriendas con varios grupos | Opción; desayuno y merienda | La opción tiene ítems de al menos 2 de estos grupos: Frutas, Cereales y derivados, Lácteos, Frutos secos y semillas. | MAN p. 22 (PDF 21): "En desayunos o meriendas se pueden incluir diferentes grupos de alimentos", y enumera frutas, pan o cereales, leche, yogur o queso, frutas secas y semillas. | S (el umbral de 2, V) |
| PA-GEN-10 | El cereal acompañado en el almuerzo y la cena | Opción; almuerzo y cena | Si la opción tiene un ítem de Cereales y derivados, tiene también uno de Verduras, Legumbres, o de Carnes, Aves, Pescados, Mariscos o Huevos. Una opción sin cereal no cuenta. | MAN p. 23 (PDF 22): "Los cereales (pastas, arroz, polenta, quinoa, fideos, otros) se pueden combinar con verduras o con legumbres" y "o con el grupo de las carnes o huevo." | S |
| PA-GEN-11 | Día de ejemplo por comida | Opción; las cuatro comidas | Un indicador por componente:<br>- Desayuno: Lácteos, Frutas frescas, Cereales y derivados.<br>- Almuerzo: Verduras, Aceites, carnes o Huevos, Frutas frescas.<br>- Merienda: Lácteos, Cereales y derivados, Frutos secos y semillas.<br>- Cena: Verduras; Cereales y derivados, Legumbres o Tubérculos y choclo; Aceites; Frutas frescas. | MAN p. 20 (PDF 19), desayuno: "1 porción de leche, yogur o queso", "1 fruta fresca", "1 porción de pan o cereales". Almuerzo: "1 porción de verduras", "1 porción de aceite", "1 porción de carne o huevo", "1 fruta fresca". MAN p. 21 (PDF 20), merienda: "1 porción de frutas secas". Cena: "1 porción de cereales, legumbres, pastas, papa, batata, choclo". | S (es un ejemplo; ver pendientes) |

PA-GEN-03 se superpone con R-03, que ya advierte si falta `vegetable` en el almuerzo o la
cena. La pauta mide lo mismo, pero en la evaluación y no en el editor. PA-GEN-11 incluye la
composición de desayuno y merienda que R-04 ya toma de MAN (DA-29); acá se mide completa,
con la fruta y las frutas secas que R-04 no exige.

## Pautas por patología

### Diabetes (DM2)

Se aplican a un paciente con Diabetes (CM-07), además de las `GEN`.

| ID | Pauta | Nivel y comidas | Indicador | Cita | Origen |
|---|---|---|---|---|---|
| PA-DM2-01 | Método del plato | Opción; almuerzo y cena | Un indicador por componente:<br>a) **Vegetal**: Verduras, sin Tubérculos y choclo ni Ajo (CM-10).<br>b) **Proteína**: Carnes, Aves, Pescados, Mariscos, Huevos o Quesos, sin Queso rallado (CM-10). Fiambres y embutidos no cuenta.<br>c) **Hidrato complejo**: Legumbres, Cereales y derivados o Tubérculos y choclo.<br>d) **Fruta**: Frutas frescas, sin Limón (CM-10).<br>e) **Grasa saludable**: Aceites, Frutas oleosas o Frutos secos y semillas.<br>Además, el porcentaje de opciones con los cinco. | p. 248: "Consiste en dividir el plato del almuerzo y cena a la mitad." Vegetal: "Una mitad siempre se ocupa con vegetales." Proteína: "alimentos que contengan proteínas (carnes, quesos magros o huevo)". Hidrato: "alimentos que contengan HC complejos (legumbres, cereales, papa, batata, mandioca o pastas)". Fruta: "Se agrega una porción de fruta." Grasa: "Se condimenta con grasas saludables en pequeñas cantidades." p. 249: "NO SE INCLUYEN en este grupos papa, batata, choclo y mandioca." Los fiambres y embutidos se evitan (p. 251). | S |
| PA-DM2-02 | Hidratos repartidos en las comidas | Opción; las cuatro comidas | Porcentaje de opciones de cada comida con `carbohydrate` (CM-01) o con un ítem de Frutas frescas sin Limón: la fruta aporta hidratos aunque no tenga la categoría (DA-42). | p. 251: "Distribuir los alimentos con hidratos de carbono en pequeñas porciones a lo largo del día, para evitar grandes volúmenes en una sola comida." | S |
| PA-DM2-03 | Hidrato acompañado de proteína | Opción; desayuno y merienda | Si la opción tiene `carbohydrate`, tiene también `protein`. En almuerzo y cena lo mide PA-DM2-01. | p. 251: "Por ejemplo en el desayuno, si se consume pan (hidratos de carbono), acompañarlo con una rodaja de queso descremado". | S |
| PA-DM2-04 | Alimentos de bajo índice glucémico | Plan | Hay un ítem de Legumbres en el plan. Además, el porcentaje de ítems de Cereales y derivados que son integrales (CM-04). Verduras y frutas ya las miden PA-GEN-03 y PA-GEN-04. | p. 250: "se recomiendan los alimentos con bajo IG (verduras, frutas, legumbres y cereales integrales)." | S |
| PA-DM2-05 | Pescados grasos y aceite de oliva | Plan | Hay un ítem de Caballa, Salmón, Atún al natural, Atún en aceite o Sardinas en lata. Hay un ítem de Aceite de oliva. Son dos indicadores. | p. 252: "pescados grasos de agua fría (ej: salmón, caballa, atún, arenques y sardinas)", "conviene aumentar su consumo a 2 o más porciones por semana", "Se aconseja completar la ingesta con grasas monoinsaturadas", con el aceite de oliva como ejemplo. | S |
| PA-DM2-06 | Edulcorante en lugar de azúcar | Opción; todas | De las opciones con un ítem de Azúcares y edulcorantes, el porcentaje en que ese ítem es Edulcorante. | p. 253: "En caso de necesitar endulzar alimentos o bebidas, utilizar edulcorantes no nutritivos como sustitutos del azúcar". | S |

- **PA-DM2-01 y R-03.** El plato pide verdura, proteína e hidrato, igual que R-03, y suma
  fruta y grasa saludable. Las cinco componentes son una pauta: no advierten.
- **PA-DM2-06 y R-05.** El azúcar ya advierte por R-05 (`moderate`). La pauta mide el lado
  positivo, el reemplazo.
- La distribución de hidratos en las 4 comidas, para quien recibe insulina o sulfonilureas
  (p. 254), no se mide aparte: la medicación no viaja a la IA (RNF-07), y PA-DM2-02 ya
  cubre las cuatro comidas.

### Hipertensión (HTA)

Se aplican a un paciente con Hipertensión (CM-07), además de las `GEN`.

| ID | Pauta | Nivel y comidas | Indicador | Cita | Origen |
|---|---|---|---|---|---|
| PA-HTA-01 | Alimentación adecuada reducida en sodio | Plan | Las mismas pautas e indicadores que PA-GEN-02, PA-GEN-04, PA-GEN-05, PA-GEN-06, PA-GEN-07 y PA-GEN-08. HTA las adopta con las mismas palabras, como parte del tratamiento. | p. 14, "Alimentación adecuada reducida en sodio", "Incluye:", y enumera "Incorporar a diario alimentos de todos los grupos.", "Consumir a diario 5 porciones de frutas y verduras en variedad de tipos y colores.", "Consumir diariamente leche, yogur o queso, preferentemente descremados.", "aumentar el consumo de pescado e incluir huevo.", "Consumir legumbres, cereales preferentemente integrales, papa, batata, choclo o mandioca." y "Consumir aceite crudo como condimento, frutas secas o semillas." | S |
| PA-HTA-02 | Condimentos en reemplazo de la sal | Plan | Hay al menos una opción con uno de estos alimentos, como ítem o como ingrediente de una receta: Limón, Vinagre, Cebolla, Cebolla de verdeo, Puerro, Ajo, Orégano, Perejil, Laurel, Pimentón, Ají molido, Pimienta, Comino, Nuez moscada, Albahaca. Además, el porcentaje de opciones de almuerzo y cena que lo tienen. La Sal con bajo contenido de sodio no cuenta: no es un reemplazo que nombre la guía. | p. 14: "Utilizar en su reemplazo: jugo de limón, vinagre, cebolla, ajo, hierbas aromáticas y especias:", y nombra, entre otros, "Orégano, Perejil (fresco o seco), Laurel, Pimentón (dulce o ahumado), Ají molido, Pimienta (negra o blanca)", "Comino" y "Nuez moscada, Albahaca". | S (la lista del catálogo, V) |

HTA no trae pautas positivas propias más allá de esas dos: no menciona el patrón DASH, el
potasio ni el método del plato. La sal fina, los fiambres y el caldo en cubo ya son reglas
(R-05).

### Celiaquía (CEL y ALG)

Se aplican a un paciente con Celiaquía (CM-07), además de las `GEN` con la salvedad de CM-03.

| ID | Pauta | Nivel y comidas | Indicador | Cita | Origen |
|---|---|---|---|---|---|
| PA-CEL-01 | Alimentos de todos los grupos, sin gluten | Plan | El mismo indicador que PA-GEN-02. Cereales y derivados cuenta solo con los alimentos de PA-CEL-02. | CEL p. 43: "Una alimentación saludable debe incluir alimentos de todos los grupos". ALG p. 7: "resulta fundamental incluir diariamente una variedad de alimentos que aporten todos los nutrientes esenciales". | S |
| PA-CEL-02 | Hidratos de fuentes sin gluten | Opción; las cuatro comidas | La opción tiene al menos un ítem de esta lista:<br>- Arroz blanco, Arroz integral, Arroz parboil, Harina de maíz (polenta), Quinoa, Amaranto, Harina de arroz, Almidón de maíz, Fécula de mandioca.<br>- Pan sin TACC, Premezcla sin TACC, Fideos sin TACC, Fideos de arroz, Galletitas sin TACC, Tostadas de arroz, Copos de maíz sin TACC.<br>- Papa, Batata, Mandioca, Choclo.<br>- Legumbres.<br>Avena sin TACC no cuenta (R-05). | CEL p. 43: "Los pacientes con Enfermedad Celíaca podrán consumir todo lo que es de origen natural, carnes vacuna, pollo, pescado, verduras y frutas frescas, legumbres, huevos, cereales sin gluten". CEL p. 44, alimentos permitidos del grupo cereales: "Granos de arroz y de maíz de cualquier marca envasados; harinas de mandioca, de papa, de soja y sus derivados envasadas y de marcas permitidas, como así también premezclas elaboradas con harinas permitidas." MAN p. 87 (PDF 81), "Cereales sin Gluten": "Harina de arroz", "harina de mandioca, quinoa, amaranto". | S (la lista del catálogo, V) |

PA-CEL-02 mide si la propuesta conserva una fuente de hidratos después de que la generación
saca los alimentos con gluten (DA-31, DA-40). Hay un conflicto: GAPA p. 27 recomienda "arroz
integral, avena, maíz, trigo burgol, cebada y centeno", y MAN p. 89 (PDF 83), la avena
arrollada para rebozar milanesas. Por CM-03, esos cereales no cuentan para un paciente
celíaco.

## Pautas por objetivo

Se revisaron solo los objetivos con una guía oficial. La relación entre `patient.goal` y un
objetivo es CM-08.

### Bajar de peso (OBE)

Se aplican al paciente cuyo objetivo es bajar de peso, además de las `GEN`. OBE p. 28: "Los
mensajes de las Guías Alimentaria para la Población Argentina (GAPA) se pueden utilizar como
base para la educación alimentaria en personas con obesidad".

| ID | Pauta | Nivel y comidas | Indicador | Cita | Origen |
|---|---|---|---|---|---|
| PA-OBE-01 | Plan basado en alimentos frescos | Plan | Cuántos de estos 7 aparecen en el plan: Verduras, Frutas, Legumbres, Frutos secos y semillas, Huevos, Pescados y Aceites. Además, el porcentaje de ítems de Cereales y derivados que son integrales (CM-04). | p. 25: "un plan de alimentación saludable basado en alimentos frescos, que incluya verduras, frutas, legumbres, frutos secos, semillas, cereales integrales (ej: arroz integral), huevo, pescados y aceites vegetales insaturados". "Evidencia moderada - FUERTE A FAVOR." | S |
| PA-OBE-02 | Pocas carnes rojas, de cortes magros | Opción; almuerzo y cena | De las opciones con `protein`, el porcentaje que tiene carnes rojas (CM-06). De los ítems de carnes rojas, el porcentaje de cortes magros (CM-06); cumple con al menos 3 de cada 4 (DA-42). | p. 25: "con bajo consumo de carnes rojas (optando por cortes magros)". p. 30, imagen: "elegir preferentemente cortes magros. Retirar la grasa visible." | S (el umbral, V) |
| PA-OBE-03 | Cuatro comidas | Plan | El mismo indicador que PA-GEN-01. | p. 32: "Planificar las comidas de la semana contemplando, en la medida de lo posible, 4 comidas diarias: desayuno, almuerzo, merienda y cena." | S |
| PA-OBE-04 | Vegetal con proteína o con hidrato | Opción; almuerzo y cena | La opción tiene `vegetable` y además `protein` o `carbohydrate`. | p. 33: "Alternativamente se puede combinar vegetales con proteínas en una de las comidas, y vegetales con hidratos de carbono en la otra comida del día." | S |
| PA-OBE-05 | Plato de almuerzo y cena | Opción; almuerzo y cena | Un indicador por componente:<br>a) **Vegetal**: Verduras, sin Tubérculos y choclo ni Ajo (CM-10).<br>b) **Proteína**: Carnes, Aves, Pescados, Mariscos, Huevos o Quesos, sin Queso rallado (CM-10). Fiambres y embutidos no cuenta (PA-OBE-07).<br>c) **Hidrato complejo**: Legumbres, Cereales y derivados o Tubérculos y choclo. Además, el porcentaje de ítems de Cereales y derivados que son integrales (CM-04).<br>d) **Fruta de postre**: Frutas frescas, sin Limón (CM-10).<br>e) **Aceite**: Aceites. Se informa, pero no entra en el cumplimiento: la fuente dice "se puede".<br>Además, el porcentaje de opciones con a, b, c y d. | p. 33, imagen, "Almuerzos y cenas:". "½ PLATO": "Verduras preferentemente de estación", y enumera acelga, brócoli, zapallo y otras. "¼ PLATO": "Alimentos que aportan proteínas: carnes (vaca, pollo, cerdo, pescado, cordero, entre otras), quesos magros o huevo." "¼ PLATO": "Alimentos que aportan hidratos de carbono complejos (legumbres, cereales o pastas de preferencia integrales, papa, batata, choclo, mandioca)." "Se puede condimentar con 1 cucharada de aceite crudo." "1 fruta como postre." | S |
| PA-OBE-06 | Desayunos y meriendas con lácteo, fruta y cereal | Opción; desayuno y merienda | Un indicador por componente:<br>- Lácteos, sin Queso rallado (CM-10).<br>- Frutas frescas, sin Limón (CM-10).<br>- Cereales y derivados.<br>Además, el porcentaje de opciones con los tres, y el porcentaje de ítems de Cereales y derivados que son integrales (CM-04) o Copos de maíz sin azúcar o sin TACC. | p. 33, imagen, "Desayunos y meriendas:", en tres columnas. "LECHE, YOGUR Y QUESO": "Leche", "Yogur natural sin agregados", "Quesos simples". "FRUTAS": "Frescas, preferentemente de estación". "CEREALES": "Pan integral", "Copos de cereal sin azúcar agregado", "Avena". | S (exigir los tres en cada opción, V) |
| PA-OBE-07 | Sin alimentos de consumo opcional | Opción; las cuatro comidas | Porcentaje de opciones con un ítem de esta lista, como ítem o como ingrediente de una receta. Cumple con 0 %.<br>- Galletitas dulces simples y Bizcochos de grasa.<br>- Fiambres y embutidos (todo el grupo).<br>- Hígado vacuno e Hígado de pollo (achuras).<br>- Manteca y Margarina.<br>- Dulce de leche, Mermelada y Mermelada reducida en azúcar.<br>- Mayonesa, Kétchup, Mostaza y Salsa de soja. | p. 31, imagen, "Grupo 6: Alimentos de consumo opcional": "Los alimentos de este grupo no deberían formar parte de la alimentación diaria". "Ejemplos de estos productos alimenticios son: galletitas dulces, saladas, amasados de pastelería (pastelitos fritos, churros, medialunas, bizcochos de grasa, facturas, otros)", "embutidos y chacinados (salchichas, chorizo, morcilla, otros), fiambres, achuras", "manteca, margarina, dulce de leche, mermeladas (industrializadas), aderezos tales como mayonesa, kétchup, mostaza, salsa golf, salsa de soja, otros." | S (la lista del catálogo y el umbral, V) |

- **PA-OBE-04 y PA-OBE-05.** El plato de p. 33 es la base, y PA-OBE-04 su alternativa
  ("Alternativamente", en la misma página): vegetal con proteína en una de las comidas
  principales, y vegetal con hidrato en la otra. Por eso PA-OBE-05 no se exige en todas las
  opciones. PA-OBE-04 choca con R-03, que pide las tres categorías en cada opción. La pauta
  no cambia la regla: una opción que la cumple sin hidrato sigue advirtiendo. Queda para
  T65.
- **PA-OBE-05 y PA-DM2-01.** Es el mismo plato, con dos diferencias: la grasa es solo aceite
  y es opcional, y la fruta es "de postre".
- **PA-OBE-06 y R-04.** R-04 pide lácteo y cereal (DA-29). La pauta suma la fruta, como
  PA-GEN-11, y prefiere cereales integrales o sin azúcar.
- **PA-OBE-07 es la única pauta que mide lo que conviene dejar afuera.** No es una regla.
  El objetivo no es una condición del paciente (CM-08) y el catálogo no tiene la patología
  Obesidad, así que el motor no advierte. La lista no incluye lo que el catálogo no tiene:
  golosinas, bebidas azucaradas, productos de copetín, carnes procesadas, helados y salsa
  golf. Hay dos choques: el Hígado vacuno es un corte magro en CM-06 y una achura acá, y
  Galletitas de agua y Chocolate pueden ser "galletitas saladas" y "golosinas". Quedan para
  T65.

### Objetivos sin guía oficial

Ningún objetivo de los casos de T41 es "bajar de peso". Para los que tienen, se revisó el
corpus:

| Caso | Objetivo (`goal`) | Pautas | Origen |
|---|---|---|---|
| C2 | Controlar glucemia | Las de DM2, por la patología del caso | V (CM-08) |
| C4 | Aumentar masa muscular | Ninguna. El corpus no tiene una guía oficial de alimentación para ganar masa muscular en adultos. La única mención, OBE p. 26, es de actividad física ("actividades de fortalecimiento muscular"). | V, sin completar |
| C5 | Mejorar la digestión | Ninguna. No hay guía oficial en el corpus. | V, sin completar |
| C1, C3, C6, C7 | Ordenar las comidas, mejorar la alimentación, comer más ordenado, mejorar hábitos | Las `GEN`, que se aplican a todos | V (CM-08) |

## No medibles con el catálogo

Lo que las fuentes recomiendan, pero el plan no puede expresar. No se evalúa en T58. Queda
en el corpus para que el RAG lo lleve a la IA como texto, y para el criterio del profesional.

| Tema | Cita | Por qué no se mide |
|---|---|---|
| Porciones y proporciones | GAPA p. 27: "Incluir 3 porciones al día de leche, yogur o queso."; DM2 p. 248, el plato dividido "a la mitad"; DM2 p. 250: "al menos 5 porciones por día de verduras o frutas"; OBE p. 30, imagen, por grupo: "4 porciones por día", "1 porción por día"; OBE p. 33, imagen: "½ PLATO", "¼ PLATO" | El plan no tiene cantidades |
| Alcohol | OBE p. 31: "Limitar el consumo de alcohol." | El catálogo no tiene bebidas alcohólicas |
| Agua | GAPA p. 26: "Tomar a diario 8 vasos de agua segura."; OBE p. 30: "Consumir 8 vasos de agua segura, prefiriendo esta a las bebidas azucaradas."; CEL p. 44: "aproximadamente 2 litros de agua potable a lo largo del día" | Cantidad. El catálogo no tiene bebidas azucaradas |
| Frecuencias semanales | GAPA p. 27: "pescado 2 o más veces por semana, otras carnes blancas 2 veces por semana y carnes rojas hasta 3 veces por semana"; GAPA p. 27: "al menos una vez por semana un puñado de frutas secas sin salar" | El plan no tiene días ni semanas |
| Colaciones | DM2 p. 251: "En presencia de hambre entre comidas realizar colaciones: fruta, yogur descremado o un puñado de frutos secos."; MAN p. 49 (PDF 46): "Preferir frutas o verduras entre comidas (media mañana o media tarde)." | El plan no tiene colaciones ni media mañana |
| Fruta entera en vez de jugo | DM2 p. 251: "evitar los jugos de frutas y consumir la fruta entera con cáscara u hollejo" | El catálogo no tiene jugos de fruta |
| Frescos, caseros o ultraprocesados | GAPA p. 26: "Elegir alimentos preparados en casa en lugar de procesados."; HTA p. 14: "Preferir alimentos naturales frescos o mínimamente procesados."; OBE p. 32: "Las recomendaciones diarias de alimentación saludable deben basarse en alimentos que no contengan sellos." | El catálogo no tiene grado de procesamiento ni sellos |
| Cocción | HTA p. 14: "Cocinar sin sal."; GAPA p. 27: "Optar por otras formas de cocción antes que la fritura."; DM2 p. 250: "pastas cocidas al dente o recalentadas" | El plan no indica la técnica, salvo dentro de la receta |
| Aceite crudo | GAPA p. 27: "Utilizar dos cucharadas soperas al día de aceite crudo." | Cantidad y forma de uso |
| Certificación y contaminación cruzada | ALG p. 8: "es recomendable que se consuman únicamente aquellos alimentos que estén autorizados como libres de gluten" | El catálogo no tiene marcas. La manipulación no es parte del plan |
| Actividad física, sueño y conducta | GAPA p. 26: "realizar al menos 30 minutos de actividad física"; OBE p. 24: "intervenciones multicomponente que incluyan terapia nutricional, actividad física e intervenciones comportamentales"; OBE p. 32: "Comer despacio, masticar bien" | No es parte del plan alimentario |
| Planes temporales y adecuación sociocultural | OBE p. 25: "alternar los planes balanceados en macronutrientes con otros planes, de manera temporal máximo 3 meses"; OBE p. 25: "se contemplen los aspectos socioculturales de la alimentación" | Decisión del profesional. Los requiere "personal capacitado" (p. 26) |

## Pendientes para la entrevista (T65)

| Pauta o criterio | Origen | Pregunta |
|---|---|---|
| Nivel plan | V | ¿Alcanza con que una pauta diaria se cumpla en al menos una opción del plan, o tiene que estar en una proporción de las opciones? |
| CM-02 | V | ¿Está bien que un ítem de grupo más amplio ("un lácteo" frente a Quesos) no cuente para una pauta de un grupo más chico? |
| PA-GEN-03, PA-DM2-01 | S | ¿Papa, batata, mandioca y choclo cuentan como hidrato y no como verdura en el plan? |
| PA-GEN-09 | V | ¿Desayuno y merienda tienen que tener al menos 2 grupos de los que propone MAN? ¿Cuántos? |
| PA-GEN-11 | S | ¿El día de ejemplo de MAN sirve como pauta por comida, o es solo una forma de distribuir los grupos? |
| PA-GEN-04, PA-DM2-01 d | S | ¿La fruta es esperable en el almuerzo y la cena, como postre? R-03 y R-04 no la exigen (DA-29). |
| CM-04 a CM-06 | V | ¿Integral se puede aproximar por la categoría `fiber`, y descremado y magro por el nombre del alimento? ¿Hace falta marcarlos en el catálogo? |
| PA-OBE-02 | V | ¿Qué proporción de opciones con carnes rojas es "bajo consumo"? |
| PA-OBE-04 | S | ¿La alternativa de vegetal con proteína o con hidrato se acepta en el almuerzo o en la cena? Hoy R-03 advierte. |
| CM-08 | V | ¿Cómo se pasa del objetivo en texto libre a un objetivo con pautas? ¿Hace falta una lista cerrada de objetivos? |
| Objetivos sin guía | V | ¿Qué pautas aplica la nutricionista a "aumentar masa muscular" y a "mejorar la digestión"? ¿Con qué fuente? |
| PA-HTA-02, PA-CEL-02 | V | ¿Las listas de alimentos del catálogo son correctas? |
| Colaciones | S | ¿La colación de DM2 p. 251 se mide en la merienda, o hace falta otra comida en el plan? |
| PA-OBE-06 | V | ¿Desayuno y merienda tienen que llevar lácteo, fruta y cereal en cada opción, o alcanza con el lácteo y el cereal de R-04? |
| PA-OBE-07 | V | ¿Galletitas de agua y Chocolate entran en el grupo de consumo opcional, como "galletitas saladas" y "golosinas"? ¿El Hígado vacuno sigue siendo un corte magro (CM-06) si OBE pone las achuras en ese grupo? |
| PA-OBE-07 | S | ¿Alcanza con medirlo en la evaluación, o hace falta una condición Obesidad con reglas `moderate` (R-05) para el grupo de consumo opcional? ¿Se mide también en la población general? |
| Versión extensa de OBE | — | Incorporarla en T46, con OCR: 117 de sus 174 páginas no tienen texto extraíble. El contenido en imagen de la versión breve (pp. 30, 31 y 33) tampoco llega al RAG sin OCR. |
| Sal con bajo contenido de sodio | V | ¿Se le ofrece a un paciente con HTA? Es cloruro de potasio, desaconsejado con enfermedad renal o con IECA, ARA II o espironolactona, y la medicación no llega al sistema. Hoy se ofrece y no cuenta para PA-HTA-02 (DA-42). |
| Ajo y queso rallado en R-03 | V | Un ítem de ajo o de queso rallado completa la verdura o la proteína de una opción en el motor. Las pautas ya no los cuentan (CM-10). ¿Hace falta que el motor tampoco los cuente? Requiere un atributo propio en el catálogo, porque cambiarles la categoría vacía los ítems de grupo (DA-30, DA-42). |
| Pacientes pediátricos | — | Las pautas y las reglas salen de guías para adultos y de la GAPA sin ajuste por edad, y C7 tiene 12 años. Faltan fuentes para niños y adolescentes (card de investigación en Trello). |
| Aderezos en DM2 | S | DM2 p. 251 nombra mayonesa, kétchup, mostaza y salsa de soja entre los productos a evitar, con la misma cita que las reglas de dulces de diabetes. Es la pregunta de R-05 sobre los ultraprocesados de DM2 en [reglas-clinicas.md](reglas-clinicas.md#pendientes-para-la-entrevista). |

## Respuestas tentativas para T65

Borrador escrito desde el rol de nutricionista, para llevar a la entrevista. Ninguna
respuesta cambia el origen: todo sigue S o V hasta que la nutricionista lo confirme o lo
corrija. Las correcciones de datos y de criterios que salieron de este borrador ya se
aplicaron ([DA-42](decisiones/correcciones-del-catalogo-por-las-pautas.md)); el resto, como
el nivel plan, PA-GEN-09 y 11, PA-OBE-04 y CM-08, sigue tentativo.

### Nivel plan

Para las pautas diarias no alcanza con una opción. El paciente elige una sola opción por
comida: si la fruta está en 1 de las 3 opciones de cena, puede no comer fruta en toda la
semana.

- **Pautas diarias** (frutas, lácteos, verduras, aceite o frutos secos: PA-GEN-04, 05 y
  08). Cumplen de forma **garantizada** si hay al menos una comida en la que todas sus
  opciones incluyen la pauta. Aparte se informa la lectura débil ("al menos una opción").
- **Pautas semanales o de variedad** (pescado, huevo, legumbres: PA-GEN-06, PA-GEN-07,
  PA-DM2-05). Cumplen con al menos una opción, porque las opciones rotan en la semana.
  Se informa además el porcentaje de opciones de almuerzo y cena que la tienen, como
  aproximación a la frecuencia: con unos 14 almuerzos y cenas por semana, "pescado 2 o más
  veces por semana" (GAPA p. 27) equivale a alrededor del 15 %.

### CM-02

De acuerdo: un ítem de grupo más amplio no cuenta. "Carnes, pescados y huevos" no asegura
pescado, porque el paciente puede elegir carne. Cuando la pauta es del grupo amplio, el
ítem de grupo sí cuenta ("un lácteo" cumple PA-GEN-05). Para CM-05, "un lácteo" queda sin
determinar: sale del denominador y no cuenta como "no descremado".

### PA-GEN-03, PA-DM2-01

Papa, batata, mandioca y choclo cuentan como hidrato. Son las verduras "C" (alrededor de
20 % de hidratos), y la GAPA las agrupa con legumbres y cereales. Arvejas frescas,
zanahoria, zapallo y remolacha siguen como verdura.

Lo que el plan cuenta como verdura y como hidrato tiene tres problemas de catálogo (ver
[observaciones](#observaciones-sobre-el-catálogo)): el ajo como verdura, la banana como el
hidrato del plato y las legumbres sin `carbohydrate`.

### PA-GEN-09

Umbral de 2 grupos para "cumple" y de 3 para "óptimo" (lácteo, cereal y fruta o frutos
secos). No exigir el lácteo en particular, por los pacientes con intolerancia, rechazo o
alimentación vegana: hoy la Bebida de soja está en Derivados de legumbres y no cuenta como
Lácteos. Con 2 grupos el indicador casi repite R-04; el umbral de 3 es el que agrega
información.

### PA-GEN-11

El día de ejemplo de MAN es una forma de distribuir los grupos, no una pauta por comida. Si
se mide como cumplimiento, contradice R-03 (carne en el almuerzo y cereal en la cena) y se
superpone con PA-GEN-09 y PA-GEN-10. Se propone sacarlo como indicador de cumplimiento o
dejarlo solo como dato descriptivo.

### PA-GEN-04, PA-DM2-01 d

La fruta de postre en el almuerzo y la cena es esperable: es una indicación habitual, y en
DM2 la pide el método del plato. No se exige en cada opción. Se mide como pauta diaria
garantizada, con la meta de tener fruta asegurada en al menos 2 comidas del día (GAPA:
"2 o 3 frutas por día"). El limón no cuenta como fruta (ver
[observaciones](#observaciones-sobre-el-catálogo)).

Aplicado en la generación ([DA-43](decisiones/orientacion-del-plan-generado.md)):
- **Fruta:** si el almuerzo o la cena no la tienen, la plantilla v4 le pide a la IA que
  sugiera fruta fresca de postre en las recomendaciones generales.
- **Opciones y porciones:** el plan empieza sus recomendaciones generales con un texto fijo.
  Aclara que las opciones son alternativas (una por comida, no tres platos ni tres tazas), y
  que almuerzo y cena se arman con el método del plato (DM2 p. 248).

### CM-04 a CM-06

La aproximación sirve como transitoria, con estas correcciones, ya aplicadas en los
criterios (DA-42):

- **Integral:** sacar Cebada perlada, porque el perlado le quita el salvado; Granola, que
  suele tener azúcar agregada y es ultraprocesada; y Salvado de avena y Salvado de trigo,
  que son un agregado y no una porción de cereal. Avena arrollada, trigo burgol, quinoa y
  amaranto están bien.
- **Descremado:** la leche parcialmente descremada cuenta. El criterio por nombre no
  detecta los quesos magros que pide DM2 p. 248: la ricota, incluso la entera, es magra.
- **Magro:** agregar Lomo de cerdo. El hígado vacuno también es magro.
- **Recomendación:** marcar los atributos en el catálogo (`whole_grain`, `low_fat`,
  `lean_cut`). El criterio por nombre falla con los alimentos privados: una "Leche light"
  no coincide.

### PA-OBE-02

Se deriva de la GAPA, que dice "carnes rojas hasta 3 veces por semana": 3 de unos 14
almuerzos y cenas, alrededor de 21 %.

- **Límite:** carnes rojas en hasta un tercio de las opciones de almuerzo y cena con
  proteína.
- **Ideal:** alrededor de un cuarto.
- **Cortes:** en un paciente que quiere bajar de peso, al menos 3 de cada 4 ítems de
  carnes rojas deberían ser cortes magros (CM-06). Asado, vacío, matambre, bondiola y carne
  picada común quedan como excepción. Aplicado en PA-OBE-02 (DA-42).

Los fiambres conviene medirlos aparte.

### PA-OBE-04

Clínicamente aceptable y habitual en planes para bajar de peso, por ejemplo una cena de
verduras con proteína, siempre que el hidrato esté en la otra comida principal (OBE p. 33).
Como los hallazgos no se descartan a mano (RF-46), esos planes van a mostrar advertencias de
R-03 permanentes, con riesgo de fatiga de alertas. Se mantiene R-03 por ahora, y queda como
el primer caso para flexibilizarla según el objetivo cuando el objetivo sea un dato
estructurado (CM-08).

### CM-08

Se prefiere una lista cerrada con selección múltiple, más un texto libre para los matices:

- Bajar de peso
- Mantener el peso
- Subir de peso
- Aumentar masa muscular
- Ordenar las comidas o mejorar hábitos
- Mejorar el tránsito intestinal
- Otro

"Controlar la glucemia" no va como objetivo, porque ya sale de la patología.

### Objetivos sin guía

- **Aumentar masa muscular:** no hay una guía oficial argentina. El criterio de consultorio
  es repartir la proteína en las cuatro comidas, incluidos desayuno y merienda, sin recortar
  hidratos. Se mide con el porcentaje de opciones de cada comida con `protein`. Sigue V; si
  la nutricionista lo confirma en la entrevista, pasa a E sin necesidad de una fuente del
  corpus. Sin una fuente, el RAG no tiene nada que llevarle a la IA sobre este objetivo:
  candidatas para T46, a verificar, los posicionamientos de ISSN sobre proteína y ejercicio
  y de ACSM, AND y DC sobre nutrición y rendimiento.
- **Mejorar la digestión:** no es un objetivo, es un síntoma, y hay que precisar qué le
  pasa al paciente.
  - Si es constipación, las pautas son fibra, agua, fruta con cáscara, legumbres e
    integrales, ya cubiertas por PA-GEN-03, 04 y 07.
  - Si es acidez o distensión, las pautas son casi todas no medibles: comidas fraccionadas
    y poco abundantes, menos frituras, cenar temprano. Más fibra y legumbres pueden empeorar
    la distensión.

  Se sugiere precisar el objetivo del caso C5.

### PA-HTA-02, PA-CEL-02

- **HTA:** la lista es correcta.
  - Agregar Cebolla de verdeo y Puerro. Aplicado (DA-42).
  - No contar la Sal con bajo contenido de sodio: no es un reemplazo que nombre la guía, y
    es cloruro de potasio, desaconsejado con enfermedad renal o con medicación que retiene
    potasio (IECA, ARA II, espironolactona). La medicación no llega al sistema (RNF-07), así
    que no se puede filtrar por eso. Aplicado en PA-HTA-02: deja de contar y deja de
    presentarse como alternativa, pero se sigue ofreciendo a la IA, porque una regla
    necesita una fuente con página y la guía no la nombra. Queda en los pendientes.
  - 38 de las 77 recetas comunes ya tienen alguno de esos condimentos (23 con cebolla o
    ajo), así que a nivel plan el indicador se cumple casi siempre. Para que discrimine,
    contar solo hierbas, especias, limón y vinagre, o medir el porcentaje por opción.
  - Fuera de la pauta: la Salsa de soja va como `moderate` para HTA (R-05, DA-42), con HTA
    p. 14 ("Reducir el uso de sal y el consumo de alimentos con alto contenido de sodio") y
    MAN p. 18, que la nombra entre los aderezos a "limitar o evitar". Los quesos duros
    (rallado, de máquina), las aceitunas y las conservas no llevan regla: el catálogo
    los considera versiones sin el sello "exceso en sodio", por una premisa del proyecto
    (DA-43, `catalogo-de-alimentos.md`).
- **CEL:** la lista es correcta como alimentos naturalmente sin gluten. En los productos
  procesados el cumplimiento depende de la certificación (CEL p. 44, "marcas permitidas"):
  - Copos de maíz: muchos tienen extracto de malta de cebada. Aplicado (DA-42): los Copos
    de maíz sin azúcar son `prohibited` para celiaquía, y la lista de PA-CEL-02 usa los
    Copos de maíz sin TACC.
  - Polenta, almidón de maíz, tostadas de arroz, quinoa y amaranto requieren certificación
    sin TACC.
  - Las legumbres a granel tienen riesgo de contaminación cruzada: indicarlas envasadas.

### Colaciones

No se miden en la merienda, que es una comida principal. DM2 indica la colación solo "en
presencia de hambre", así que es opcional. No se agrega otra comida al plan. En pacientes
con insulina o sulfonilureas, la colación la pauta el profesional como recomendación. Se
mantiene en "No medibles".

### OBE pp. 30, 31 y 33

Leídas a mano el 13/11/2026. Sumaron PA-OBE-05 a PA-OBE-07 y dos citas de respaldo (PA-GEN-03
y PA-OBE-02). Las preguntas que abren quedan en los
[pendientes](#pendientes-para-la-entrevista-t65), sin respuesta tentativa.

### Observaciones sobre el catálogo

Errores que distorsionaban las reglas o los indicadores con los datos del seed. Quedaron
resueltos en [DA-42](decisiones/correcciones-del-catalogo-por-las-pautas.md), salvo el ajo y
el queso rallado en R-03, que se resuelven solo en las pautas.

| Problema | Efecto que tenía | Corrección |
|---|---|---|
| Limón está en Frutas frescas | 9 recetas de almuerzo y cena lo llevan como condimento (½ unidad) y cumplían la fruta de PA-GEN-04 y PA-DM2-01 d sin tener fruta | CM-10: no cuenta como porción de fruta. Sigue en Frutas frescas, así una condición sobre Frutas lo alcanza |
| Ajo tiene `vegetable` | Un ítem de ajo completaba la verdura de R-03 y de PA-GEN-03 | CM-10 lo excluye de Verduras en las pautas. Conserva `vegetable`: con `complementary`, el ítem de grupo "Bulbos, raíces y tallos" se quedaba sin categoría (DA-30). En R-03 sigue completando la verdura, y queda como pregunta para T65. La cebolla queda como está: "Arroz integral con lentejas y cebolla caramelizada" lleva 2 unidades |
| Banana tenía `carbohydrate` | R-03 y R-04 la aceptaban como el hidrato del plato | Banana queda solo con `fiber`. Los dos licuados con banana suman avena arrollada, y "Yogur con banana y semillas de lino" suma copos de maíz sin TACC. PA-DM2-02 cuenta `carbohydrate` o Frutas frescas |
| Legumbres sin `carbohydrate` | Una opción de legumbre con verdura advertía por falta de hidrato | Las Legumbres secas y en conserva suman `carbohydrate`. Tofu, soja texturizada, bebida de soja y harina de garbanzo no cambian |
| Queso rallado tiene `protein` | "Fideos integrales con brócoli, ajo y queso rallado" pasaba R-03 sin una porción de proteína | Esa receta suma ricota, y CM-10 excluye el queso rallado de Lácteos y de la proteína en las pautas. Conserva `protein`: con `complementary`, los ítems de grupo Lácteos y Quesos se quedaban sin categoría, y "un lácteo y pan" advertía en el desayuno (DA-30). En R-03 sigue completando la proteína, y queda como pregunta para T65 |
| Copos de maíz sin azúcar | Se ofrecían a un celíaco como hidrato sin gluten, y suelen tener malta | Regla `prohibited` para celiaquía, como la de Malta. Alimento nuevo Copos de maíz sin TACC, que usa "Yogur descremado con copos de maíz y mango" |

## Fundamento

- [Pautas de adecuación nutricional v1](decisiones/pautas-de-adecuacion-v1.md) (DA-41)
- [Correcciones del catálogo por las pautas de adecuación](decisiones/correcciones-del-catalogo-por-las-pautas.md) (DA-42)
- [Orientación del plan generado](decisiones/orientacion-del-plan-generado.md) (DA-43)
- [Composición esperada por tipo de comida](decisiones/composicion-esperada-por-tipo-de-comida.md) (DA-29)
- [Reglas por patología v1](decisiones/reglas-por-patologia-v1.md) (DA-36)
- [Restricciones de lo que se ofrece a la IA](decisiones/restricciones-de-lo-ofrecido-a-la-ia.md) (DA-40)
- [Alcance del Incremento 3](decisiones/alcance-del-incremento-3.md) (DA-38, que actualiza DA-13)
- [Reglas clínicas del motor determinístico](reglas-clinicas.md)
