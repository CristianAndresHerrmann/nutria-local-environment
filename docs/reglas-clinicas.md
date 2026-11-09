# Reglas clínicas del motor determinístico

Catálogo de las reglas que valida el motor de `nutria-backend/src/domain/reglas/`, cada
una con su origen. Si una regla no tiene su ficha acá, no existe. Cómo se agrega o se
cambia una regla está en el skill `reglas-clinicas-deterministicas`
(`.agents/skills/reglas-clinicas-deterministicas/SKILL.md`).

## Propósito y alcance

El motor valida el plan alimentario y produce hallazgos, que el nutricionista revisa en
el editor. Vive en el backend, separado del servicio de IA y del RAG: la IA propone y el
motor valida. La información nutricional general no se mezcla con las restricciones
clínicas obligatorias. Lo que conviene ofrecer (pautas positivas por patología, objetivo y
población general) no son reglas: está en [Pautas de adecuación](pautas-de-adecuacion.md)
y no genera hallazgos (DA-41).

| Requerimiento | Cómo lo cubre el motor |
|---|---|
| RF-43 | Validación determinística contra las condiciones del paciente y las reglas de sus patologías. Lo contraindicado bloquea (R-01, R-02, R-05). Lo que una regla por patología solo modera, advierte (R-05). |
| RF-44 | La composición esperada de cada comida genera una advertencia que no bloquea la aprobación (R-03, R-04). Una receta en una comida que no es la suya también advierte (R-06). |
| RF-46 | Los hallazgos se recalculan y se reemplazan en cada guardado (`planValidation.service.revalidate`). No se descartan a mano. |
| RF-47 | Las reglas por patología son datos en `clinical_rule`, con fuente bibliográfica obligatoria (R-05). |

| Riesgo del Plan | Mitigación |
|---|---|
| 1. Planes inseguros generados por la IA | Toda propuesta pasa por el motor antes de aprobarse. Lo que el motor no puede validar (ítem no reconocido) bloquea. |
| 2. Reglas clínicas incompletas o no validadas | Cada regla tiene ficha con condición, restricción, alcance, comportamiento esperado y origen. Las S y V quedan listadas para la entrevista. |
| 15. Datos o fuentes no validados | Las restricciones dependen del catálogo controlado y de reglas con fuente, nunca de información generada o recuperada. |
| 18. Información inventada por la IA | El motor valida solo datos estructurados. El asistente de desarrollo tampoco agrega reglas sin origen documentado. |

## Origen de una regla

| Código | Origen | Referencia obligatoria |
|---|---|---|
| **E** | Entrevista de validación con nutricionistas | Fecha y registro de la entrevista |
| **P** | Plan de proyecto y requerimientos | RF o riesgo del Plan, y la DA que la baja a código |
| **S** | Supuesto con fuente bibliográfica | DA que lo registra, y la fuente (obra y páginas) |
| **V** | Criterio pendiente de validación, sin fuente clínica propia | DA que lo adopta, y la pregunta para la entrevista |

- Las cuatro se implementan. S y V quedan además en
  [Pendientes para la entrevista](#pendientes-para-la-entrevista).
- El origen lo documenta una persona: un requerimiento, una decisión registrada o la
  entrevista. Un asistente de IA no crea orígenes.
- Cuando la entrevista confirma una S o una V, pasa a E, y se actualizan la ficha y la
  DA. Si la corrige, se registra la decisión nueva y se cambia la regla.

## Catálogo

### R-01 Condición directa

| | |
|---|---|
| Condición | Alergia, intolerancia o rechazo del paciente, por alimento o por grupo. |
| Restricción | El alimento de la condición, o cualquier alimento del grupo o de sus subgrupos. |
| Alcance | Ítem de alimento (comunes y privados). Ingrediente de un ítem de receta. Ítem de grupo, si su grupo es el de la condición o un subgrupo. |
| Comportamiento | `direct_condition`, `blocking`. Un hallazgo por cada par ítem-condición, con `optionItemId` y `patientConditionId`. En una receta, el mensaje nombra los ingredientes alcanzados. |
| Origen | **P**: RF-37, RF-43; riesgos 1 y 2. Mismo criterio de "comprendido en" que el filtro de recetas (RF-28, DA-23). Bajada a código en DA-30. |
| Implementación | `directCondition.js`; alcances en `src/utils/recipeConditionFilter.js` (`conditionScopes`, `scopeCoversFood`). |
| Pruebas | `directCondition.test.js` y `validatePlan.test.js` ("condición directa dentro del plan"). |

Criterios de aplicación:

- **Un ítem de grupo más amplio que la condición no dispara.** Por ejemplo, "una fruta"
  con alergia a la banana. Origen **V** (DA-30). Desde T54, ese grupo no se le ofrece a la
  IA (DA-40): solo puede llegar al plan si el profesional lo agrega a mano.
- Las preferencias no disparan. Origen **P** (DA-23).
- Las condiciones de tipo patología no disparan acá. Pasan por R-05.

Casos de prueba:

| Dispara | No dispara |
|---|---|
| Alimento directo de una alergia | Una preferencia, por alimento o por grupo |
| Alimento del grupo de una intolerancia | Una patología (es R-05) |
| Alimento de un subgrupo del grupo de un rechazo | Un alimento o una receta sin nada restringido |
| Ingrediente de una receta, nombrado en el mensaje | Un ítem de grupo más amplio que la condición |
| Varios ingredientes alcanzados por la misma condición dan un solo hallazgo | Un ítem no reconocido (tiene R-02) |
| Alimento privado del nutricionista, por condición directa y por su grupo | |
| Ítem de grupo comprendido en el grupo de la condición | |
| Un hallazgo por cada condición que alcanza al ítem | |

Si un alimento o una receta no está en el catálogo, es un error y el ítem no se saltea.

### R-02 Ítem no reconocido

| | |
|---|---|
| Condición | Ninguna: depende del ítem. |
| Restricción | Un ítem de tipo `unrecognized`, es decir, texto de la propuesta que no se asoció al catálogo. |
| Alcance | El ítem. |
| Comportamiento | `unrecognized_item`, `blocking`, con `optionItemId`. Se levanta reemplazando el ítem por un alimento, un grupo o una receta. Dejarlo con su texto es posible en el borrador, pero el bloqueo se mantiene. |
| Origen | **P**: RF-43, que valida solo datos estructurados; riesgos 1 y 18. El hallazgo propio viene de DA-26 y la severidad, de DA-30. |
| Implementación | `validatePlan.js` (`unrecognizedItemFinding`). |
| Pruebas | `validatePlan.test.js`, "ítem no reconocido": dispara un bloqueo sobre el ítem; no dispara con ítems del catálogo; no aporta categorías, así que la opción además puede quedar incompleta. |

### R-03 Composición de almuerzo y cena

| | |
|---|---|
| Condición | Ninguna: se aplica a toda opción de almuerzo o de cena. |
| Restricción | La opción tiene que cubrir `vegetable`, `protein` y `carbohydrate`. |
| Alcance | La opción, con las categorías que aportan sus ítems. |
| Comportamiento | `composition`, `warning`, con `optionId` y sin ítem. El mensaje numera la opción dentro de su comida y nombra lo que falta. Nunca bloquea. |
| Origen | **P**: RF-26, RF-44 (DA-29). |
| Implementación | `mealComposition.js` (`MEAL_COMPOSITION`, `missingCategories`); `validatePlan.js` (`compositionFinding`). |
| Pruebas | `mealComposition.test.js`; `validatePlan.test.js`, "composición": opción completa sin hallazgos; opción incompleta con advertencia sobre la opción; cada opción se evalúa por separado. |

Criterio de aplicación: **las tres categorías se exigen en todas las opciones.** Origen
**V** (DA-29): la GAPA reparte la proteína y el cereal entre almuerzo y cena.

### R-04 Composición de desayuno y merienda

| | |
|---|---|
| Condición | Ninguna: se aplica a toda opción de desayuno o de merienda. |
| Restricción | La opción tiene que cubrir `protein` (el lácteo) y `carbohydrate` (pan o cereales). La fruta y las frutas secas no se exigen. |
| Alcance | Igual que R-03. |
| Comportamiento | Igual que R-03. |
| Origen | **S**: DA-29. Fuente: *Manual para la aplicación de las Guías Alimentarias para la Población Argentina*, Ministerio de Salud, pp. 20-22. |
| Implementación | `mealComposition.js` (`BREAKFAST_AND_SNACK`). |
| Pruebas | `validatePlan.test.js`: un ítem de grupo ("un lácteo") y pan completan el desayuno. |

### Criterio compartido de R-03 y R-04: categorías de un ítem

| Tipo de ítem | Categorías que aporta | Origen |
|---|---|---|
| Alimento | La copia `food_categories` del ítem, no el catálogo actual | **P** (DA-16) |
| Receta | La unión de las categorías de sus ingredientes | **V** (DA-30) |
| Grupo | La intersección de las categorías de los alimentos vigentes del grupo y de sus subgrupos | **V** (DA-30) |
| No reconocido | Ninguna | **P** (DA-30) |

Implementación en `itemCategories.js` y pruebas en `itemCategories.test.js`. Una receta
con las categorías de sus ingredientes también se prueba en `validatePlan.test.js`.

### R-05 Reglas por patología

| | |
|---|---|
| Condición | Una patología del paciente. |
| Restricción | Un alimento o un grupo, según cada fila vigente de `clinical_rule`. Ver la [matriz cargada](#matriz-cargada). |
| Alcance | Igual que R-01: ítem de alimento (comunes y privados), ingrediente de un ítem de receta, e ítem de grupo si su grupo es el de la regla o un subgrupo. |
| Comportamiento | `pathology_rule`, con `optionItemId` y `clinicalRuleId`. Si la regla es `prohibited`, el hallazgo es `blocking`; si es `moderate`, es `warning` y no impide aprobar. Hay un hallazgo por cada par de ítem y regla, y el mensaje nombra la patología y la fuente. Además, las reglas `prohibited` sacan su alimento o su grupo del catálogo que se envía a la IA (DA-31), y también los grupos que los contienen (DA-40). |
| Origen | **S** en cada fila (DA-36): la fuente bibliográfica (obra y páginas) está en la regla (`clinical_rule.source`, obligatoria por `clinical_rule_source_chk`, RF-47) y en la matriz. La entrevista de validación no se hizo, y la matriz pasa a E cuando una nutricionista la confirme. |
| Implementación | `pathologyRule.js` (`pathologyRuleScopes`, `pathologyRuleFindings`); los alcances, en `ruleScopes` y `scopeCoversFood` de `src/utils/recipeConditionFilter.js`. Las reglas las carga `planValidation.service.loadInputs` (`clinicalRule.repository.findActiveForPathologies`). Los datos están en `scripts/seed-data/clinical-rules.js`. |
| Pruebas | `pathologyRule.test.js`, `validatePlan.test.js` ("reglas por patología dentro del plan"), `planValidation.service.test.js`, `planGeneration.service.test.js` y `app.int.test.js` (T35). Los datos se prueban en `clinical-rules.test.js` y `seed.int.test.js`, y los casos de T41 en `clinical-cases.rules.test.js`. |

Criterios de aplicación:

- **La severidad sigue la redacción de la fuente.** Si dice "exclusión", "descartar" o "no
  usar", la regla es `prohibited`. Si dice "evitar", "limitar", "reducir" o "no se
  recomienda", es `moderate`. Origen **S** (DA-36).
- **Lo que se modera advierte y no bloquea** (RF-44). Origen **S** (DA-36).
- **Qué alimento del catálogo contiene trigo, avena, cebada o centeno** es un criterio del
  desarrollador, sobre el catálogo de T28. Origen **V** (DA-36, DA-32).
- **Un ítem de grupo más amplio que la regla no dispara**, igual que en R-01. Por ejemplo,
  "Panificados y harinas" en celiaquía, porque las reglas son por alimento. Origen **V**
  (DA-30). Desde T54 la generación ya no ofrece esos grupos a un paciente con la patología
  (DA-40), y `clinical-cases.rules.test.js` lo controla para C1 con los datos del seed.
- Las reglas de una patología que el paciente no tiene no se evalúan. Una alergia y una
  regla sobre el mismo alimento dan dos hallazgos.

Casos de prueba:

| Dispara | No dispara |
|---|---|
| Alimento de una regla `prohibited`: bloqueo, con la regla y su fuente | Regla de una patología que el paciente no tiene |
| Alimento de una regla `moderate`: advertencia | Paciente sin patologías, aunque tenga otras condiciones |
| Alimento del grupo de una regla, y de un subgrupo | Alimento sin regla del mismo grupo (Pan sin TACC) |
| Alimento privado del nutricionista en el grupo de una regla | Ítem de grupo más amplio que la regla |
| Ingrediente de una receta, nombrado en el mensaje | Receta sin ingredientes alcanzados |
| Ítem de grupo comprendido en el grupo de la regla | Ítem no reconocido (tiene R-02) |
| Un hallazgo por cada regla que alcanza al ítem | Regla dada de baja (`active = false`) |

#### Matriz cargada

Fuentes, cada cita verificada contra el texto de su página en el corpus `datos-rag/`:

- **CEL:** Ministerio de Salud de la Nación. *Guía de práctica clínica sobre diagnóstico
  y tratamiento de la enfermedad celíaca en el primer nivel de atención*. 2016.
- **ALG:** ANMAT / INAL. *Guía de recomendaciones para un menú libre de gluten seguro*,
  2.ª ed. 2023.
- **DM2:** Ministerio de Salud de la Nación. *Guía de Práctica Clínica Nacional sobre
  Prevención, Diagnóstico y Tratamiento de la Diabetes Mellitus Tipo 2*. 2019.
- **HTA:** Ministerio de Salud de la Nación. *Guía de Práctica Clínica Nacional.
  Prevención, Diagnóstico y Tratamiento de la Hipertensión Arterial — versión breve*. 2026.
- **GAPA:** Ministerio de Salud de la Nación. *Guías alimentarias para la población
  argentina*. 2016.
- **MAN:** Ministerio de Salud de la Nación. *Manual para la aplicación de las Guías
  Alimentarias para la Población Argentina*. 2018. Página impresa (la del PDF va entre
  paréntesis).

| Patología | Alimento o grupo | Severidad | Fuente | Origen |
|---|---|---|---|---|
| Celiaquía | Alimentos con trigo, avena, cebada o centeno: Pan francés, Pan integral, Pan lactal blanco, Pan lactal integral, Pan de salvado, Pan árabe, Harina de trigo, Harina integral, Pan rallado, Tapas de tarta, Tapas de empanada, Fideos secos, Fideos integrales, Fideos frescos, Masa para lasaña, Avena arrollada, Salvado de avena, Salvado de trigo, Trigo burgol, Cebada perlada, Galletitas de agua, Galletitas integrales, Galletitas de salvado, Galletitas dulces simples, Bizcochos de grasa, Granola, Copos de maíz sin azúcar (suelen llevar malta de cebada; DA-42), Malta (una regla por alimento) | `prohibited` | CEL pp. 35 y 43: "la exclusión de cuatro cereales (trigo, avena, cebada y centeno), sus derivados y productos que los contengan de por vida" | S (la lista de alimentos, V) |
| Celiaquía | Avena sin TACC | `moderate` | CEL p. 43: "en Argentina no se recomienda que los pacientes consuman alimentos con avena por el peligro de contaminación cruzada"; ALG p. 15: "se recomienda evitar su utilización en el menú LG, o bien, informar acerca de su presencia" | S |
| Diabetes | Azúcar, Azúcar mascabo | `moderate` | DM2 p. 245: azúcares agregados ("azúcar de mesa o sacarosa"), "es recomendable evitar su consumo"; p. 253: edulcorantes no nutritivos como sustitutos del azúcar | S |
| Diabetes | Mermelada, Dulce de leche, Galletitas dulces simples, Bizcochos de grasa | `moderate` | DM2 p. 251: "Evitar los productos alimenticios procesados y ultraprocesados con altas cantidades de grasas, azúcares y/o sal: galletitas dulces, […] bizcochos de grasa, […] dulce de leche, mermeladas (industrializadas)" | S |
| Hipertensión | Caldo en cubo | `prohibited` | HTA p. 14: "No usar saborizantes en polvo o caldos deshidratados como condimentos" | S |
| Hipertensión | Sal fina | `moderate` | HTA p. 8: "reducido en sodio: 5 gramos de sal"; p. 14: "Reducir el uso de sal" | S |
| Hipertensión | Grupo Fiambres y embutidos | `moderate` | GAPA p. 26: "Los fiambres, embutidos y otros alimentos procesados […] contienen elevada cantidad de sodio"; HTA p. 14: "Reducir […] el consumo de alimentos con alto contenido de sodio" | S |
| Hipertensión | Salsa de soja | `moderate` | HTA p. 14: "Reducir el uso de sal y el consumo de alimentos con alto contenido de sodio"; MAN p. 18 (PDF 17), alimentos ultraprocesados, "Se recomienda limitar o evitar su consumo": "Aderezos: mayonesa, salsa golf, mostaza, kétchup, salsa de soja" (DA-42) | S (que sea un alimento alto en sodio, V) |

En la base, la fuente de cada regla es la cita corta con páginas (por ejemplo, "MSAL, GPC
enfermedad celíaca (2016), pp. 35 y 43"), y es la que muestra el hallazgo.

#### Casos de T41

El resultado esperado de los casos clínicos (`scripts/seed-data/clinical-cases.js`) se
prueba en `clinical-cases.rules.test.js`. Ese test corre el motor sobre un mismo plan de
prueba, con los datos reales del seed. El plan tiene los siguientes ítems:

- Pan francés, Pan sin TACC y Avena sin TACC.
- El grupo Cereales y derivados.
- La receta Milanesa de carne al horno con ensalada (con pan rallado y huevo).
- Azúcar, Edulcorante, Caldo en cubo y Sal con bajo contenido de sodio.
- Jamón cocido y el grupo Fiambres y embutidos.
- Huevo, Leche y Lentejas.

| Caso | Condición | Hallazgos esperados |
|---|---|---|
| C1 | Celiaquía | Pan francés y la milanesa (por el pan rallado): bloqueo. Avena sin TACC: advertencia. |
| C2 | Diabetes | Azúcar: advertencia. |
| C3 | Hipertensión | Caldo en cubo: bloqueo. Jamón cocido y el grupo Fiambres y embutidos: advertencia. |
| C4 | Alergia al grupo Huevos | Huevo y la milanesa: bloqueo (R-01). |
| C5 | Intolerancia a la leche | Leche: bloqueo (R-01). |
| C6 | Sin condiciones | Ninguno. |
| C7 | Rechazo del grupo Legumbres | Lentejas: bloqueo (R-01). |

Ninguno dispara con Pan sin TACC, Edulcorante, Sal con bajo contenido de sodio, ni con el
grupo Cereales y derivados.

### R-06 Receta fuera de su tipo de comida

| | |
|---|---|
| Condición | Ninguna: depende del ítem y de la comida que lo contiene. |
| Restricción | Un ítem de receta en una comida que no está entre los tipos de comida de la receta (`recipe_meal_type`). |
| Alcance | Ítems de receta, de la propuesta de la IA y de la edición manual. |
| Comportamiento | `meal_type_mismatch`, `warning`, con `optionItemId`. El mensaje nombra la receta, la comida y los tipos de comida para los que se indica. Nunca bloquea. |
| Origen | **P**: RF-27 (la receta declara sus tipos de comida), RF-28 y RF-43; riesgo 1. Bajada a código en DA-40 (T54). La severidad es **V** (DA-40): no es un riesgo clínico, como la composición. |
| Implementación | `recipeMealType.js` (`recipeMealTypeFinding`), cableado en `validatePlan.js` después de las reglas por patología. Migración `agrega_hallazgo_de_receta_fuera_de_su_comida`. |
| Pruebas | `recipeMealType.test.js`; `validatePlan.test.js` ("receta fuera de su tipo de comida (R-06)" y `finding_origin_chk`); `plan.repository.int.test.js` (CHECK contra Postgres). |

Casos de prueba:

| Dispara | No dispara |
|---|---|
| Receta de almuerzo y cena en el desayuno | Receta en uno de sus tipos de comida |
| Receta con una condición y fuera de su comida: los dos hallazgos, la condición primero | Alimentos, grupos e ítems no reconocidos |

Una receta que no está en el catálogo es un error, y el ítem no se saltea.

## Pendientes para la entrevista

Lo que la entrevista confirma o corrige. La entrevista de T35 no se hizo, así que la
matriz de R-05 quedó con origen S y también se revisa ahí.

| Regla | Origen | Pregunta |
|---|---|---|
| R-01 | V | ¿Un ítem de grupo que incluye un alimento restringido ("una fruta" con alergia a la banana) tiene que avisar? (DA-30) |
| R-03 | V | ¿Almuerzo y cena exigen siempre verdura, proteína e hidrato en cada opción? (DA-29) |
| R-03 | V | Un ítem de ajo o de queso rallado completa la verdura o la proteína, aunque se use en cantidad de condimento. ¿Tiene que dejar de contar? Requiere un atributo propio en el catálogo: cambiarles la categoría deja sin categoría a los ítems de grupo Lácteos, Quesos y "Bulbos, raíces y tallos" (DA-30, DA-42). |
| R-04 | S | ¿Desayuno y merienda llevan lácteo y pan o cereales? ¿Hace falta exigir fruta? (DA-29) |
| R-03, R-04 | V | ¿Está bien que un ítem de grupo solo cubra las categorías que comparten todos sus alimentos, y que una receta cubra las de cualquiera de sus ingredientes? (DA-30) |
| R-05 | S | ¿Se confirma la matriz de celiaquía, diabetes e hipertensión? ¿Las severidades son correctas (lo que se excluye bloquea; lo que se modera advierte)? (DA-36) |
| R-05 | V | ¿La lista de alimentos con trigo, avena, cebada o centeno es correcta (por ejemplo, la malta y los copos de maíz sin azúcar, que suelen llevarla)? (DA-36, DA-42) |
| R-05 | S | ¿La avena sin TACC se modera, se excluye o se permite? (DA-36) |
| R-05 | S | ¿Diabetes alcanza solo azúcares y dulces, o también la lista de ultraprocesados de la GPC DM2, pp. 251-252 (grasas, fiambres, aderezos)? (DA-36) |
| R-05 | V | Un ítem de grupo amplio ("un pan") no avisa por celiaquía. ¿Tiene que avisar? Desde T54 la IA ya no lo propone; queda el caso del profesional que lo agrega a mano. (DA-30, DA-36, DA-40) |
| R-06 | V | ¿Una receta en una comida que no es la suya tiene que advertir, bloquear o nada? (DA-40) |
| R-05 | — | Fuera de la matriz: productos industrializados que dependen del listado ALG (fiambres, condimentos, salsa de soja). ¿Se agregan? (DA-36) |
| R-05 | V | Hipertensión ya modera la salsa de soja (DA-42). ¿También quesos duros (rallado, de máquina), aceitunas y conservas? ¿Y la sal con bajo contenido de sodio, que es cloruro de potasio? |

## Fundamento

- [Composición esperada por tipo de comida](decisiones/composicion-esperada-por-tipo-de-comida.md) (DA-29)
- [Motor de reglas v1](decisiones/motor-de-reglas-v1.md) (DA-30)
- [Reglas por patología v1](decisiones/reglas-por-patologia-v1.md) (DA-36)
- [Restricciones de lo que se ofrece a la IA y receta fuera de su comida](decisiones/restricciones-de-lo-ofrecido-a-la-ia.md) (DA-40)
- [Pautas de adecuación nutricional v1](decisiones/pautas-de-adecuacion-v1.md) (DA-41), que no son reglas
- [Correcciones del catálogo por las pautas de adecuación](decisiones/correcciones-del-catalogo-por-las-pautas.md) (DA-42)
- [Ajustes al modelo del Anexo A — Incremento 3](decisiones/ajustes-al-modelo-del-anexo-a-incremento-3.md) (DA-26, DA-27)
- [Registro de decisiones del Incremento 2](registro-de-decisiones-incremento-2.md) (DA-16, DA-23)
- [Registro de decisiones del Incremento 3](registro-de-decisiones-incremento-3.md)
