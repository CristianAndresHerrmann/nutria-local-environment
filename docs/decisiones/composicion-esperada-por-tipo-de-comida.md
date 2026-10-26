# Composición esperada por tipo de comida

**Fecha:** 26/10/2026
**Tarea relacionada:** T27

## Contexto

RF-26 pide la composición esperada de cada tipo de comida, y RF-44 una advertencia
cuando una opción del plan no la cumple. La Tabla 3 del registro del Incremento 2 la
dejó como configuración del Incremento 3, no como tabla. Se expresa sobre las
categorías nutricionales del catálogo (RF-25): `protein`, `carbohydrate`, `fiber`,
`vegetable`, `healthy_fat` e `infusion`.

Almuerzo y cena se conocen por RF-26: verdura, proteína e hidrato. Desayuno y
merienda iban a definirse en la entrevista de validación con la nutricionista, que
todavía no se hizo. Lo único que se sabe es que pueden compartir alimentos y recetas.
Sin una composición para desayuno y merienda, la mitad de las comidas del plan no
recibe ninguna advertencia de composición en el Incremento 3.

La referencia nacional es el *Manual para la aplicación de las Guías Alimentarias para
la Población Argentina* (Ministerio de Salud de la Nación), pp. 20 a 22:

- **Desayuno:** una porción de leche, yogur o queso, una fruta fresca y una porción de
  pan o cereales, preferentemente integrales.
- **Merienda:** una porción de leche, yogur o queso, una porción de pan o cereales y
  una porción de frutas secas.
- **Desayunos o meriendas:** frutas, pan o cereales, lácteos, frutas secas y semillas
  "se pueden incluir".

## Alternativas consideradas

- **Dejar desayuno y merienda sin definir hasta la entrevista** (sin advertencias). No inventa nada, pero el motor llega a T37 sin poder advertir en dos
  de las cuatro comidas, y la entrevista no tiene fecha.
- **Una composición distinta para cada una, igual al ejemplo del manual** (lácteo,
  cereal y fruta en el desayuno; lácteo, cereal y frutas secas en la merienda). Es la
  más fiel al manual, pero:
  - El catálogo no tiene una categoría para la fruta. Manzana y naranja son `fiber`,
    igual que la avena, el arroz integral y las legumbres. Exigir `fiber` no garantiza
    una fruta.
  - Separa dos comidas que comparten alimentos y recetas.
  - El mismo manual presenta los grupos como opcionales.
- **Agregar categorías `dairy` y `fruit` al catálogo.** Distingue el lácteo de otras
  proteínas, y la fruta de otras fuentes de fibra. Pero es un cambio de esquema y del
  seed, fuera del alcance de una configuración.

## Decisión

Desayuno y merienda comparten una composición: `protein` (el lácteo) y
`carbohydrate` (pan o cereales). Es lo común a ambas en el manual, y es un supuesto
a confirmar en la entrevista de validación. Almuerzo y cena quedan como dice RF-26.
Vive en `nutria-backend/src/domain/reglas/mealComposition.js`.

## Consecuencias

- El motor de reglas (T37) advierte en las cuatro comidas. Son advertencias
  (RF-44): el profesional las revisa y nunca bloquean la aprobación, así que un
  supuesto conservador acota el riesgo.
- `protein` también la cubre un huevo o un fiambre: la advertencia detecta la falta de
  proteína, no la falta de lácteo. Lo mismo pasa con `carbohydrate` y una fruta como
  la banana.
- No se advierte por la falta de fruta. La GAPA la reparte entre comidas y
  colaciones, y el catálogo no la distingue.
- El manual distribuye la proteína y el cereal entre almuerzo y cena (en su ejemplo,
  carne en el almuerzo y cereal o legumbre en la cena). RF-26 pide los tres en
  ambas. Se mantiene RF-26, que es un requerimiento, y la diferencia se lleva a la
  entrevista.
- Un ítem de grupo ("un lácteo") no trae categorías, y uno de receta tampoco. El motor
  de reglas las deriva (T34, DA-30, [`motor-de-reglas-v1.md`](motor-de-reglas-v1.md)):
  la receta aporta las de sus ingredientes, y el grupo las que comparten todos sus
  alimentos vigentes.
- **Para la entrevista de validación:**
  - Confirmar o corregir la composición de desayuno y merienda.
  - Decidir si la fruta es esperable en alguna comida.
  - Decidir si almuerzo y cena exigen siempre las tres categorías.
