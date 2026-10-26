---
name: reglas-clinicas-deterministicas
description: Cómo funciona y cómo se extiende el motor de reglas determinístico del backend de NutrIA (hallazgos de un plan alimentario). Usar al tocar nutria-backend/src/domain/reglas/, el servicio planValidation o cualquier cálculo de hallazgos, y antes de agregar o cambiar una regla.
---

# Motor de reglas determinístico — backend

El motor vive en `nutria-backend/src/domain/reglas/`. Valida un plan alimentario y
produce **hallazgos**, que el nutricionista revisa en el editor. Las decisiones que lo
fundamentan son DA-29 y DA-30 (`docs/decisiones/composicion-esperada-por-tipo-de-comida.md`
y `docs/decisiones/motor-de-reglas-v1.md` en el repo raíz).

## Principios (no se negocian)

- El motor vive en el **backend**, nunca en el servicio de IA. La IA propone y el motor
  valida. Ninguna sugerencia se aprueba sin pasar por el motor y por la revisión del
  nutricionista.
- **No se inventan reglas clínicas.** Una regla nueva necesita un requerimiento, una
  decisión registrada o la entrevista de validación con nutricionistas. Si falta, se
  pregunta y no se codifica un supuesto.
- Las reglas por patología son **datos** en `clinical_rule` (patología → alimento o grupo,
  severidad, fuente bibliográfica). No son código, ni texto en un prompt. Se cargan en T35.
- La información nutricional general (bibliografía, RAG) no se mezcla con las
  restricciones clínicas obligatorias.
- Los hallazgos **se recalculan y se reemplazan** en cada guardado (RF-46). No se
  descartan a mano ni llevan marca de resuelto. Un bloqueo se levanta corrigiendo lo que lo
  produce.

## Estructura

| Archivo | Qué hace |
|---|---|
| `validatePlan.js` | Entrada del motor. `validatePlan({ plan, conditions, catalog }) → Hallazgo[]` y `validationResult(findings)` (para `ai_generation.validation_result`). |
| `directCondition.js` | Regla de condición directa (RF-37, RF-43). |
| `itemCategories.js` | Categorías que cada tipo de ítem aporta a la composición. |
| `mealComposition.js` | Configuración de la composición esperada por tipo de comida (T27). Se edita acá, versionada. |
| `catalogIndex.js` | Indexa el catálogo de entrada y corta si falta un id. |

Fuera de la carpeta:

- `src/utils/recipeConditionFilter.js`: `conditionScopes` y `scopeCoversFood` definen
  qué está comprendido en una condición. Son el **único** criterio, compartido con el
  filtro de recetas de RF-28 (DA-23). No dupliques esa lógica.
- `src/services/planValidation.service.js`: `revalidate(planId, nutritionistId)` arma las
  entradas y llama a `plan.repository.replaceFindings`, que borra y crea en una
  transacción. La generación y el editor lo llaman después de cada guardado. No hay
  endpoint propio.

### Entradas

- `plan`: con la forma de `PLAN_INCLUDE` (comidas → opciones → ítems).
- `conditions`: las del paciente, de `patientConditionRepository.findAllForPatient`.
- `catalog`: `{ foods, groups, recipes }`:
  - `foods`: alimentos comunes y privados del nutricionista, **incluidos los dados de
    baja** (`foodRepository.findAllVisible`).
  - `groups`: todos los grupos.
  - `recipes`: las recetas del nutricionista, incluidas las no disponibles.

El motor es una función pura: no usa Prisma, no hace I/O y no lee la hora. Para la misma
entrada da la misma salida, en el mismo orden.

### Hallazgo

Un hallazgo tiene las columnas de `Finding`, salvo `id`, `planId` y `detectedAt`:
`{ type, severity, optionId, optionItemId, patientConditionId, message }`. Debe cumplir
`finding_origin_chk`:

| Tipo | Severidad | Referencias | Dispara cuando |
|---|---|---|---|
| `unrecognized_item` | `blocking` | ítem | El ítem es `unrecognized`. |
| `direct_condition` | `blocking` | ítem + condición | El alimento, un ingrediente de la receta o el grupo del ítem está comprendido en una alergia, intolerancia o rechazo (por alimento, o por grupo con sus subgrupos). Genera un hallazgo por cada par de ítem y condición. |
| `composition` | `warning` | opción (sin ítem) | Falta una categoría esperada para la comida. |
| `pathology_rule` | (T35) | ítem + regla | La define T35, a partir de `clinical_rule`. |

Lo que **no** dispara:

- Las preferencias.
- Las condiciones de tipo patología. Esas pasan por `clinical_rule`.
- Un ítem de grupo más amplio que la condición. Por ejemplo, "una fruta" con alergia a la
  banana (DA-30).

## Agregar o cambiar una regla

1. **Fundamento primero.** Ubicá el requerimiento o la decisión que pide la regla. Si
   introduce un criterio nuevo (severidad, alcance, excepción), registrá una decisión con
   `docs/decisiones/_plantilla.md`, más su fila DA en el registro del incremento. Si
   depende de la entrevista, se deja anotado y no se implementa.
2. **Función pura** en `src/domain/reglas/<regla>.js`. Recibe lo que necesita, ya indexado,
   y devuelve hallazgos con la forma de arriba. Reutilizá `conditionScopes` y
   `scopeCoversFood` para todo lo que sea "comprendido en".
3. **Compatibilidad con `finding_origin_chk`.** Si el tipo de hallazgo o sus referencias
   no entran en el CHECK actual, el cambio es de modelo: seguí el skill
   `prisma-migraciones`.
4. **Cableado** en `validatePlan.js`, respetando el orden. Por ítem: no reconocido, después
   condiciones y después reglas por patología. La composición va al final de cada opción.
5. **Pruebas** en `<regla>.test.js` y en `validatePlan.test.js`, siempre casos que disparan
   **y** casos que no. Cubrí alimento directo, grupo y subgrupo, ingrediente de receta,
   alimento privado y lo que explícitamente no dispara. Corré `npm run test:backend` desde
   la raíz (skill `pruebas-vitest`).
6. Si cambia lo que el servicio necesita cargar, actualizá
   `planValidation.service.js` y su test con repositorios fake.
7. Agregá la entrada en el `CHANGELOG.md` de `nutria-backend` (no en el README).

## Cambiar la composición esperada

La configuración vive en `mealComposition.js` (DA-29). Cambiarla es una decisión de
dominio y no se hace por iniciativa propia. Corresponde con una tarea propia o con el
resultado de la entrevista de validación, y se actualiza DA-29.

## Prohibido

- Decidir una restricción clínica en el servicio de IA o en un prompt.
- Leer `nutritionistId` del cliente. Siempre viene del token.
- Saltear en silencio un ítem cuyo alimento o receta no está en el catálogo. Se corta con
  un error.
- Acumular hallazgos, o borrar uno suelto a pedido del usuario.
