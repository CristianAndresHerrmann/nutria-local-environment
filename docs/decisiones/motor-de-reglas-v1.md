# Motor de reglas v1: condiciones directas, ítem no reconocido y composición

**Fecha:** 26/10/2026
**Tarea relacionada:** T34

## Contexto

T34 implementa la primera versión del motor determinístico del backend
(`nutria-backend/src/domain/reglas/`). Toma el plan, las condiciones del paciente y el
catálogo, y devuelve hallazgos. Incluye las reglas que no dependen de la matriz clínica:

- Condición directa (RF-37, RF-43).
- Ítem no reconocido (DA-26).
- Composición esperada (RF-44, configuración de T27, DA-29).

Las reglas por patología quedan para T35.

Al bajar esas reglas a código quedaron cinco puntos sin definir:

- **Severidad del ítem no reconocido.** DA-26 le da un hallazgo propio, pero no dice si
  bloquea. También deja que el profesional "lo deje con su texto".
- **Categorías de un ítem de grupo o de receta.** La composición se calcula sobre las
  categorías de los ítems. DA-29 dejó para el motor cómo se derivan las de un ítem de grupo
  ("un lácteo") y las de una receta.
- **Ítem de grupo más amplio que la condición.** Por ejemplo, "una fruta" con alergia a
  la banana, o "Carnes" con rechazo de "Pescados". RF-28 no tiene este caso, porque solo
  filtra recetas por sus ingredientes.
- **Granularidad del hallazgo.** Puede haber uno por ítem, o uno por cada condición que
  alcanza al ítem.
- **Catálogo de entrada.** Un ítem o un ingrediente puede referenciar un alimento que se
  dio de baja después.

## Alternativas consideradas

- **Ítem no reconocido como advertencia.** El profesional podría aprobar con el texto
  libre. Pero el motor no lo validó, y un alérgeno escrito como texto pasaría sin control
  (riesgo 1).
- **Categorías del ítem de grupo por unión** (cualquier categoría de algún alimento del
  grupo). Da menos advertencias, pero puede dar por cubierta una categoría que el alimento
  elegido no tiene.
- **Ítem de grupo sin categorías.** Es lo más simple, pero "un lácteo" en el desayuno
  daría siempre una advertencia de proteína.
- **Advertencia cuando el grupo del ítem incluye algo restringido.** Es más cauto, pero
  es una regla que ningún requerimiento define. Además, el profesional elige el alimento
  concreto al revisar el plan.
- **Un hallazgo por ítem**, con todas las condiciones en el mensaje. Así se pierde la FK a
  la condición (`patient_condition_id`), que borra el hallazgo cuando se quita la condición
  de la ficha.

## Decisión

- **El ítem no reconocido bloquea.** No se puede validar hasta reemplazarlo por un
  alimento, un grupo o una receta del catálogo. Modifica DA-26: dejarlo con su texto sigue
  siendo posible en el borrador, pero mantiene el bloqueo.
- **Categorías que aporta cada ítem a la composición:**
  - Alimento: la copia `food_categories` del ítem (DA-16).
  - Receta: la unión de las categorías de sus ingredientes.
  - Grupo: la **intersección** de las categorías de los alimentos vigentes del grupo y de
    sus subgrupos. Es lo que el grupo garantiza, sea cual sea el alimento elegido.
  - No reconocido: ninguna.
- **Un ítem de grupo dispara una condición directa solo si su grupo está dentro del grupo
  de la condición** (es el mismo grupo o un subgrupo). Si la condición es más específica que
  el ítem, no hay hallazgo.
- **Un hallazgo por cada par de ítem y condición.** En una receta, el mensaje nombra los
  ingredientes alcanzados.
- **El catálogo de entrada incluye los alimentos dados de baja**, comunes y privados del
  nutricionista. Para la intersección de un grupo se usan solo los vigentes.
- El criterio de "comprendido en una condición" es uno solo para el filtro de recetas
  (RF-28) y para el motor (DA-23): `conditionScopes` y `scopeCoversFood`, en
  `src/utils/recipeConditionFilter.js`.

## Consecuencias

- Una propuesta de la IA con ítems no reconocidos queda con bloqueos. Eso sostiene el
  indicador de propuestas sin bloqueos del IA2, y mide si el catálogo alcanza (DA-21).
- La configuración de composición de T27 no cambia. La intersección puede dar alguna
  advertencia de más en grupos heterogéneos, pero nunca da por cubierta una categoría que
  falta. Una advertencia no bloquea (RF-44).
- **Un ítem de grupo amplio no avisa de lo que excluye.** Esto queda como riesgo
  aceptado: el profesional revisa el plan antes de aprobarlo (RF-48). Se lleva a la
  entrevista de validación.
- **Los hallazgos se reemplazan en cada revalidación** (`planValidation.service.revalidate`
  y `plan.repository.replaceFindings`).
  - Quitar una condición de la ficha borra sus hallazgos por cascada.
  - El servicio lee el plan fuera de la transacción de escritura. Si hay dos guardados
    simultáneos del mismo plan, pueden quedar los hallazgos del estado anterior hasta el
    próximo guardado. Se acepta, porque cada plan lo edita un solo profesional.
- Un alimento o una receta que no está en el catálogo del nutricionista es una
  inconsistencia de datos. El motor corta con un error y no saltea el ítem.
- **Para la entrevista de validación:** si un ítem de grupo que incluye un alimento
  restringido debe avisar.
