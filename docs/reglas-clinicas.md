# Reglas clínicas del motor determinístico

Catálogo de las reglas que valida el motor de `nutria-backend/src/domain/reglas/`, cada
una con su origen. Si una regla no tiene su ficha acá, no existe. Cómo se agrega o se
cambia una regla está en el skill `reglas-clinicas-deterministicas`
(`.agents/skills/reglas-clinicas-deterministicas/SKILL.md`).

## Propósito y alcance

El motor valida el plan alimentario y produce hallazgos, que el nutricionista revisa en
el editor. Vive en el backend, separado del servicio de IA y del RAG: la IA propone y el
motor valida. La información nutricional general no se mezcla con las restricciones
clínicas obligatorias.

| Requerimiento | Cómo lo cubre el motor |
|---|---|
| RF-43 | Validación determinística contra las condiciones del paciente y las reglas de sus patologías. Lo contraindicado bloquea (R-01, R-02, R-05). |
| RF-44 | La composición esperada de cada comida genera una advertencia que no bloquea la aprobación (R-03, R-04). |
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
  con alergia a la banana. Origen **V** (DA-30).
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

### R-05 Reglas por patología (T35)

| | |
|---|---|
| Condición | Una patología del paciente. |
| Restricción | Un alimento o un grupo, según cada fila de `clinical_rule`. |
| Comportamiento | `pathology_rule`, con `optionItemId` y `clinicalRuleId`. T35 define cómo se pasa de `prohibited`/`moderate` a la severidad del hallazgo. |
| Origen | Una por fila. La fuente bibliográfica es obligatoria (`clinical_rule_source_chk`, RF-47). Se espera origen **E**: la matriz sale de la entrevista de validación. |
| Implementación | Pendiente (T35). Las reglas son datos, no código ni texto de un prompt. |

Fuera del motor, la generación (DA-31) ya usa `clinical_rule`: las reglas `prohibited`
vigentes de las patologías del paciente sacan esos alimentos del catálogo que se envía
a la IA.

Matriz cargada (la completa T35; no se precargan reglas):

| Patología | Alimento o grupo | Severidad | Fuente | Origen |
|---|---|---|---|---|
| | | | | |

## Pendientes para la entrevista

Lo que la entrevista confirma o corrige. La matriz de patologías de R-05 también sale de
ahí, pero es parte de T35.

| Regla | Origen | Pregunta |
|---|---|---|
| R-01 | V | ¿Un ítem de grupo que incluye un alimento restringido ("una fruta" con alergia a la banana) tiene que avisar? (DA-30) |
| R-03 | V | ¿Almuerzo y cena exigen siempre verdura, proteína e hidrato en cada opción? (DA-29) |
| R-04 | S | ¿Desayuno y merienda llevan lácteo y pan o cereales? ¿Hace falta exigir fruta? (DA-29) |
| R-03, R-04 | V | ¿Está bien que un ítem de grupo solo cubra las categorías que comparten todos sus alimentos, y que una receta cubra las de cualquiera de sus ingredientes? (DA-30) |

## Fundamento

- [Composición esperada por tipo de comida](decisiones/composicion-esperada-por-tipo-de-comida.md) (DA-29)
- [Motor de reglas v1](decisiones/motor-de-reglas-v1.md) (DA-30)
- [Ajustes al modelo del Anexo A — Incremento 3](decisiones/ajustes-al-modelo-del-anexo-a-incremento-3.md) (DA-26, DA-27)
- [Registro de decisiones del Incremento 2](registro-de-decisiones-incremento-2.md) (DA-16, DA-23)
- [Registro de decisiones del Incremento 3](registro-de-decisiones-incremento-3.md)
