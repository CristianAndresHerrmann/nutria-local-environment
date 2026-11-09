# Correcciones del catálogo por las pautas de adecuación

**Fecha:** 09/11/2026
**Tarea relacionada:** T57

## Contexto

Las respuestas tentativas para T65, en
[`pautas-de-adecuacion.md`](../pautas-de-adecuacion.md#respuestas-tentativas-para-t65),
encontraron errores de datos en el catálogo del seed. Esos errores distorsionaban las reglas
de composición (R-03, R-04), la matriz por patología (R-05) y los indicadores de las pautas:

- **Banana:** tenía `carbohydrate`, así que R-03 y R-04 la aceptaban como el hidrato del
  plato. 3 recetas comunes de desayuno solo tenían ese hidrato.
- **Legumbres:** no tenían `carbohydrate`. Una opción de legumbre con verdura advertía por
  falta de hidrato, aunque la GAPA las agrupa con los cereales y PA-DM2-01 las cuenta como
  hidrato complejo.
- **Ajo** (`vegetable`) y **Queso rallado** (`protein`) completaban la verdura o la
  proteína de una opción, aunque se usan en cantidad de condimento.
- **Copos de maíz sin azúcar:** se ofrecían a un celíaco, pero los comunes suelen llevar
  extracto de malta de cebada.
- **Limón:** 9 recetas de almuerzo y cena lo llevan como condimento, y con eso cumplían la
  pauta de fruta.
- **CM-04 a CM-06:** contaban Cebada perlada y Granola como integrales, y no contaban la
  ricota ni el lomo de cerdo como magros.
- **Salsa de soja:** el catálogo la presenta como alimento identificable para HTA, pero no
  tenía regla.

## Alternativas consideradas

- **Mover Ajo y Limón a "Sal, especias y hierbas".**
  - Ventaja: arregla a la vez las categorías y la pertenencia a un grupo.
  - Desventaja: una condición sobre Verduras o sobre Frutas (por ejemplo, alergia a los
    cítricos) deja de alcanzarlos. Además rompe la taxonomía por origen (DA-21).
- **Cambiar categorías y criterios de medición, sin mover grupos (elegida).**
  - Ventaja: R-01 sigue alcanzando a esos alimentos por su grupo.
  - Desventaja: hace falta un criterio aparte (CM-10) para que no cuenten como porción.
- **Renombrar "Copos de maíz sin azúcar" a "Copos de maíz sin TACC".** Descartada: el seed
  actualiza por nombre y no borra, así que el nombre viejo quedaría activo en las bases ya
  cargadas, sin regla.
- **Agregar `carbohydrate` a todas las frutas.** Descartada: la fruta pasaría a completar
  el hidrato del plato en R-03 y R-04.
- **Pasar Ajo y Queso rallado a `complementary`.** Se aplicó y se revirtió el mismo día.
  Un ítem de grupo aporta solo las categorías que comparten todos sus alimentos (DA-30), así
  que Lácteos, Quesos y "Bulbos, raíces y tallos" se quedaban sin categoría, y "un lácteo y
  pan" advertía en el desayuno. Los tests no lo detectaban porque el motor se probaba con un
  catálogo propio.
- **Que los alimentos `complementary` no cuenten en la intersección de un grupo.**
  Descartada: es un cambio del criterio de DA-30, y "Grasas y aceites" pasaría a aportar
  `healthy_fat` aunque incluye manteca y crema.
- **Cargar también quesos duros, aceitunas y la sal con bajo contenido de sodio en la
  matriz de HTA.** Queda para T65. La sal baja en sodio no tiene una fuente con página que
  la nombre, y una regla sin fuente no se carga (RF-47).

## Decisión

1. **Categorías del catálogo:**
   - Banana: solo `fiber`. Ninguna fruta aporta `carbohydrate`.
   - Legumbres secas y en conserva: suman `carbohydrate`.
   - Ajo y Queso rallado conservan `vegetable` y `protein`. No cuentan como porción solo
     en las pautas (CM-10).
   - Alimento nuevo: Copos de maíz sin TACC (`carbohydrate`).
2. **Matriz de R-05:**
   - Copos de maíz sin azúcar, `prohibited` para celiaquía (CEL pp. 35 y 43). Que lleven
     malta es criterio V.
   - Salsa de soja, `moderate` para hipertensión (HTA p. 14 y MAN p. 18). Origen S; que
     sea un alimento alto en sodio es V.
3. **Recetas comunes:** 5 cambian de ingredientes sin cambiar de nombre, para cumplir la
   composición de DA-29 (detalle en `recetas-basicas.md`).
4. **Criterios de medición de las pautas:**
   - CM-04: sin Cebada perlada, Granola ni salvados.
   - CM-05: con la Ricota; un ítem de grupo sale del denominador.
   - CM-06: con Lomo de cerdo e Hígado vacuno.
   - CM-10 nuevo: Limón, Ajo y Queso rallado no cuentan como porción.
   - PA-DM2-02 cuenta también la fruta.
   - PA-HTA-02 suma Cebolla de verdeo y Puerro, y no cuenta la sal baja en sodio.
   - PA-CEL-02 usa los copos sin TACC.
   - PA-OBE-02 cumple con 3 de cada 4 cortes magros.

Todo sigue con origen S o V hasta la entrevista de T65.

## Consecuencias

- **Sin migración.** El seed reescribe las categorías por nombre (`setCategories`) y agrega
  el alimento y las reglas nuevas. Staging necesita correr el seed a mano después del
  despliegue.
- **Los planes ya guardados no cambian**, porque sus ítems guardan una copia de las
  categorías (DA-16). Un ítem nuevo de banana deja de completar el hidrato, y uno de
  legumbre lo completa.
- **Ajo y queso rallado en R-03:** un ítem de ajo o de queso rallado sigue completando la
  verdura o la proteína en el motor. Corregirlo necesita un atributo propio en el catálogo,
  y queda como pregunta para T65.
- **Generación:** a un paciente celíaco ya no se le ofrecen los copos de maíz comunes, ni
  las 2 recetas que los siguen usando, ni los 2 licuados que ahora llevan avena arrollada.
  Las recetas comunes de desayuno candidatas para C1 pasan de 11 a 7 (de 17 a 13 con las
  privadas), todavía por encima del mínimo de 5 por comida que controla
  `clinical-cases.rules.test.js`. A un hipertenso la salsa de soja se le sigue ofreciendo,
  y advierte.
- **Línea de base:** cambian las categorías y las recetas que viajan a la IA, así que la
  medición sin RAG del 07/11/2026 deja de ser comparable 1 a 1 y se vuelve a medir con el
  catálogo corregido.
- **Cobertura de los tests de datos:**
  - `catalogs.test.js` controla las frutas sin `carbohydrate` y las legumbres secas con él.
  - `clinical-cases.rules.test.js` corre el motor con las categorías reales del seed: un
    lácteo, un queso, un bulbo y una legumbre como ítems de grupo completan la opción, y la
    fruta no completa el hidrato.
- Completa DA-36 y DA-41.
