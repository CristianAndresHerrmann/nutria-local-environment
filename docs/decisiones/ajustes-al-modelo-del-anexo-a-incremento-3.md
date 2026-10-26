# Ajustes al modelo de datos del Anexo A — Incremento 3

**Fecha:** 26/10/2026
**Tarea relacionada:** T32

## Contexto

T32 incorpora en una sola migración (`agrega_planes_validacion_y_generaciones_ia`) las
entidades que la Tabla 3 del registro del Incremento 2 asigna al Incremento 3: PLAN,
PLAN_COMIDA, OPCION, OPCION_ITEM, REGLA_CLINICA, HALLAZGO, GENERACION_IA y
RECETA.plan_origen_id (diferido por DA-19). Al llevar el diagrama A.3 a un esquema
implementable aparecieron cuatro cosas:

- El diagrama mezcla atributos del Incremento 3 con los de aprobación y versionado del
  Incremento 4.
- No prevé ítems que la propuesta de la IA nombra pero el catálogo no reconoce. El IA2 los
  dejó como punto de diseño abierto.
- No guarda la copia del alimento que exige DA-16.
- Anota restricciones ("excluyente con grupo") que Prisma no expresa.

## Alternativas consideradas

- **Implementar el A.3 tal cual, con los atributos del Incremento 4.** Evita una migración
  posterior. En contra, deja columnas sin uso ni prueba durante un incremento. Tampoco
  resuelve los ítems no reconocidos ni la copia del catálogo.
- **Implementar solo lo del Incremento 3, ajustar lo que piden los requerimientos y
  registrar los desvíos.** Los estados de aprobación se agregan en el Incremento 4 con un
  `ALTER TYPE ... ADD VALUE` y columnas opcionales, sin reescribir datos.

## Decisión

Se implementa lo del Incremento 3 con los ajustes siguientes.

### Ajustes respecto del Anexo A

- **Aislamiento del plan en la base (RNF-05).**
  - PLAN lleva `nutritionist_id` además de `patient_id`, para que el repositorio filtre en
    la misma tabla, igual que con los pacientes.
  - Una FK compuesta `(patient_id, nutritionist_id) → patient(id, nutritionist_id)` hace
    que la base rechace un plan cuyo nutricionista no es el de su paciente. Prisma la
    expresa (con `@@unique([id, nutritionistId])` en `Patient`), así que no genera drift.
- **Recomendaciones como texto (DA-16).**
  - Se conservan los cuatro textos del Anexo: generales, contextuales, clínicas y
    temáticas.
  - Las recomendaciones del catálogo que se eligen se copian en
    `clinical_recommendations`. Desde ahí el profesional las edita (RF-40), y una
    corrección posterior del catálogo no cambia el plan.
- **Fraccionamiento (RF-36).** PLAN_COMIDA.indicaciones pasa a `instructions`: es la
  indicación textual de fraccionamiento dirigida al paciente.
- **Copia del catálogo en el ítem (DA-16).** Un ítem de alimento guarda `food_name` y
  `food_categories`, y uno de grupo guarda `food_group_name`. La FK se mantiene para la
  validación. Las categorías se copian porque las usa el control de composición (RF-44).
- **Tipo de ítem explícito.** OPCION_ITEM suma `type`, con cuatro valores:
  - `food`: alimento del catálogo.
  - `food_group`: la "referencia genérica" del Anexo, como "una fruta".
  - `recipe`: el Anexo pone la receta en OPCION.receta_id. Como ítem, una opción puede
    combinar una receta con alimentos sueltos, y la validación recorre una sola lista.
  - `unrecognized`: ver DA-26.
- **Ítem no reconocido (DA-26).** Lo que la propuesta nombra y no se resuelve contra el
  catálogo se guarda con el texto original. No se descarta ni se fuerza a un alimento
  parecido.
  - Genera un hallazgo `unrecognized_item`, porque el motor no puede validarlo (RF-43
    opera sobre datos estructurados).
  - El profesional lo reemplaza por un alimento, un grupo o una receta, o lo deja con su
    texto.
- **HALLAZGO.**
  - El tipo suma `unrecognized_item`. Los valores del Anexo se traducen: condicion_paciente
    → `direct_condition`, regla_clinica → `pathology_rule`, composicion_incompleta →
    `composition`.
  - Los hallazgos se recalculan en cada revalidación (RF-46), así que no llevan `resuelto`.
    Un bloqueo se levanta corrigiendo lo que lo produjo, y entonces deja de existir.
  - `generacion_ia_id` también se omite: el resultado de la validación inicial queda en la
    generación (ver abajo).
- **REGLA_CLINICA.**
  - Severidad `prohibited` | `moderate`, como en el Anexo (prohibido | a_moderar). Qué
    severidad de hallazgo produce cada una lo decide el motor de reglas (T37).
  - Es un catálogo común, como las patologías (DA-15), con baja lógica (`active`).
  - El atributo `version` se difiere: no hay requerimiento que pida versionar una regla, y
    el hallazgo ya referencia la regla vigente al validar.
- **GENERACION_IA.**
  - Se agregan los datos que pide RNF-11: proveedor, modelo, versión de plantilla,
    duración y tokens (cuando el proveedor los informa).
  - `result` registra cómo terminó la llamada: `success`, `invalid_response`,
    `provider_error` o `timeout`, los errores recuperables que prevé el IA2.
  - `validation_result` conserva el resultado de validar la propuesta tal como llegó
    (RF-41). Sostiene el indicador del IA2: proporción de propuestas sin hallazgos
    bloqueantes. Los hallazgos del plan no sirven para eso, porque cambian con cada
    edición.
  - `plan_id` es obligatorio: el borrador se crea antes de invocar a la IA (T34), así una
    generación fallida también queda registrada y aislada a través del plan.
  - `contexto_enviado` es `jsonb`: es el contexto anonimizado tal como se envió.
- **Fechas.** PLAN lleva `created_at` / `updated_at`, con el mismo criterio que PACIENTE,
  CONSULTA y RECETA. HALLAZGO.detectado_en → `detected_at`; GENERACION_IA.generado_en →
  `generated_at`.

### Atributos del Anexo que no se incorporan ahora

| Atributo | Destino |
|---|---|
| PLAN.plan_anterior_id, version, fecha_aprobacion, fecha_primera_exportacion | Incremento 4: versionado, vínculo entre planes, aprobación y exportación (RF-42, RF-48, RF-49). |
| PLAN.confirmacion_datos_faltantes_en | Incremento 4: la confirmación se pide antes de aprobar (RF-45, DA-22). |
| PLAN.validado_en | Con el motor de reglas (T37), si el editor necesita mostrar cuándo se validó. |
| PLAN.consulta_id, PLAN.nota_interna | Sin requerimiento del Incremento 3 que los use; se agregan si una tarea los necesita. |
| PLAN_COMIDA.orden | No hace falta: el orden es el del enum `meal_type`. |
| PLAN_COMIDA.observacion_actual | Sin requerimiento del Incremento 3 que lo use. |
| OPCION.receta_id | Reemplazado por el ítem de tipo `recipe`. |
| OPCION.es_vianda | Sin definir en el Incremento 3. El IA2 trata la comida fuera del hogar como una comida más (RF-34); se resuelve con la tarea que la incorpore. |
| REGLA_CLINICA.version | Ver arriba. |
| HALLAZGO.generacion_ia_id, resuelto | Ver arriba. |
| GENERACION_IA.tipo, receta_id | Con la generación de recetas (RF-39), que define si una generación de receta cuelga de un plan. |
| GENERACION_IA.modificaciones_profesional | Incremento 4: son las modificaciones previas a la aprobación (RF-41). |

### Restricciones que no expresa Prisma

Son `CHECK` en el SQL de la migración. La del ítem se valida también en
`src/schemas/plan.schema.js` (`optionItemSchema`), para que la API responda `400` con el
campo puntual. Las del hallazgo y la generación las escribe el backend, no el cliente.
Un `CHECK` que evalúa a `NULL` se cumple, así que cada condición se arma para que nunca
dé `NULL`.

| Constraint | Regla |
|---|---|
| `option_item_type_chk` | `food`: alimento y copia del nombre. `food_group`: grupo y copia del nombre. `recipe`: receta. `unrecognized`: texto original no vacío. En cada tipo, el resto de las referencias y copias quedan vacías. |
| `clinical_rule_target_chk` | Exactamente uno de alimento o grupo (el Anexo anota "excluyente"). Una regla sin ninguno no restringe nada. |
| `clinical_rule_source_chk` | Fuente bibliográfica no vacía (RF-47). |
| `finding_origin_chk` | `direct_condition`: ítem y condición del paciente. `pathology_rule`: ítem y regla. `composition`: opción, sin ítem. `unrecognized_item`: ítem. La severidad no se fija en la base: la decide el motor de reglas. |
| `ai_generation_metrics_chk` | Duración y tokens no negativos. |
| `ai_generation_response_chk` | La respuesta cruda existe cuando el proveedor respondió (`success`, `invalid_response`). |
| `ai_generation_validation_chk` | El resultado de la validación solo existe si la respuesta se pudo usar (`success`). |

Las reglas que cruzan tablas no entran en un `CHECK` y quedan a cargo de los servicios
(T34, T37):

- El ítem o la opción de un hallazgo son del mismo plan.
- La receta de un ítem es del nutricionista del plan.
- El alimento de un ítem es común o privado de ese nutricionista.
- Una receta con `origin_plan_id` es del mismo nutricionista que el plan.
- La condición de un hallazgo es del paciente del plan.

### Política de borrado

- Lo que pertenece al plan (comidas, opciones, ítems, hallazgos y generaciones) se borra
  con él, y el plan con su paciente, igual que el resto de los datos del paciente
  (DA-19).
- Un hallazgo se borra con la condición del paciente que lo origina. Se recalculan en cada
  revalidación (RF-46), y con `RESTRICT` no se podría quitar una condición de la ficha.
- Las referencias a catálogos (alimento, grupo, receta, patología, regla) son `RESTRICT`:
  se dan de baja de forma lógica.
- Si se borra el plan, la receta generada para él queda con `origin_plan_id` en `NULL`.

### Correspondencia con el Anexo A

| Anexo A | Tabla y columnas | Modelo en Prisma |
|---|---|---|
| PLAN | `plan` (`patient_id`, `nutritionist_id`, `status` = estado, `situational_diagnosis`, `introduction`, `objectives`, `general_recommendations`, `contextual_recommendations`, `clinical_recommendations`, `thematic_recommendations`, `created_at`, `updated_at`) | `Plan` |
| PLAN_COMIDA | `plan_meal` (`plan_id`, `meal_type` = tipo, `instructions` = indicaciones) | `PlanMeal` |
| OPCION | `meal_option` (`plan_meal_id`, `order`, `description`) | `MealOption` |
| OPCION_ITEM | `option_item` (`option_id`, `order`, `type`, `food_id`, `food_name`, `food_categories`, `food_group_id`, `food_group_name`, `recipe_id`, `original_text`) | `OptionItem` |
| REGLA_CLINICA | `clinical_rule` (`pathology_id`, `food_id`, `food_group_id`, `severity`, `source` = fuente_bibliografica, `active` = vigente) | `ClinicalRule` |
| HALLAZGO | `finding` (`plan_id`, `option_id`, `option_item_id`, `clinical_rule_id`, `patient_condition_id`, `type`, `severity`, `message`, `detected_at`) | `Finding` |
| GENERACION_IA | `ai_generation` (`plan_id`, `context_sent` = contexto_enviado, `raw_response` = respuesta_obtenida, `validation_result` = resultado_validacion, `provider`, `model`, `template_version`, `duration_ms`, `input_tokens`, `output_tokens`, `result`, `generated_at`) | `AiGeneration` |
| RECETA.plan_origen_id | `recipe.origin_plan_id` | `Recipe.originPlanId` |

| Enum del Anexo A | Valores |
|---|---|
| PLAN.estado | borrador → `draft` (aprobado, en el Incremento 4) |
| PLAN_COMIDA.tipo | El enum `meal_type` existente: desayuno → `breakfast`, almuerzo → `lunch`, merienda → `afternoon_snack`, cena → `dinner` |
| OPCION_ITEM.tipo (nuevo) | `food`, `food_group`, `recipe`, `unrecognized` |
| REGLA_CLINICA.severidad | prohibido → `prohibited`, a_moderar → `moderate` |
| HALLAZGO.tipo | condicion_paciente → `direct_condition`, regla_clinica → `pathology_rule`, composicion_incompleta → `composition`, nuevo → `unrecognized_item` |
| HALLAZGO.severidad | bloqueo → `blocking`, advertencia → `warning` |
| GENERACION_IA.resultado_validacion | sin_hallazgos → `no_findings`, con_advertencias → `warnings`, con_bloqueos → `blocking` |
| GENERACION_IA.result (nuevo) | `success`, `invalid_response`, `provider_error`, `timeout` |

## Consecuencias

- `schema.prisma` es el modelo lógico vigente de los Incrementos 1 a 3, y las tablas de
  arriba son la referencia entre el Anexo A y el esquema real.
- La integridad de los ítems y los hallazgos no depende de la aplicación. Si un servicio
  se saltea la validación Zod, el error lo da la base.
- El Incremento 4 agrega los estados de aprobación al enum y sus columnas, todas
  opcionales o con valor por defecto, sin reescribir planes existentes.
- Los `CHECK` y la FK compuesta se verificaron a mano contra Postgres en T32: un INSERT
  válido y uno inválido por combinación, dentro de una transacción revertida. Pasan a
  pruebas de integración (`*.int.test.js`) cuando exista esa infraestructura.
- T34 debe crear el borrador antes de invocar a la IA. T37 debe recalcular los hallazgos
  del plan en cada revalidación, sin acumularlos.
