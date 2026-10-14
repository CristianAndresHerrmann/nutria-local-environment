# Ajustes al modelo de datos del Anexo A — Incremento 2

**Fecha:** 14/10/2026
**Tarea relacionada:** T14

## Contexto

T14 incorpora en una sola migración (`agrega_datos_clinicos_consultas_y_catalogos`) las
entidades del Anexo A del IA1 que el IA2 (Tabla 3) asigna al Incremento 2: los datos clínicos y
contextuales del paciente, PATOLOGIA, PACIENTE_CONDICION, CONSULTA, MEDICION, HABITO_PAUTADO,
GRUPO_ALIMENTO, ALIMENTO, ALIMENTO_CATEGORIA, RECETA, RECETA_INGREDIENTE, RECETA_TIPO_COMIDA y
RECOMENDACION_CLINICA. Al llevar los diagramas A.1 y A.2 a un esquema implementable aparecieron
huecos frente a los requerimientos (RF-06, RF-08) y reglas que el diagrama anota pero Prisma no
expresa.

## Alternativas consideradas

- **Implementar el Anexo A tal cual.** No habría desvíos que justificar. En contra: RF-08
  quedaría cubierto en parte (no hay dónde registrar "si cocina"), los hábitos de RF-06 quedarían
  mezclados en un solo texto y las restricciones anotadas en el diagrama ("excluyente con
  grupo", "solo si tipo es patologia") no tendrían quién las hiciera cumplir.
- **Ajustar el modelo, aplicar las restricciones en la base y en la aplicación, y registrar los
  desvíos.** El esquema responde a lo que piden los requerimientos y el desvío queda documentado
  para el IA3.

## Decisión

Se ajusta el modelo. Ajustes respecto del Anexo A:

- **`cooks` (si cocina, RF-08):** columna nueva, `Boolean` opcional. El Anexo A solo tiene
  `gusta_cocinar`, y no es lo mismo cocinar que querer hacerlo: alguien puede no cocinar y querer
  aprender, o cocinar por obligación.
- **Hábitos (RF-06):** `habitos` se divide en cinco columnas de texto opcionales, una por ítem
  del requerimiento: actividad física, tabaco, alcohol, horarios de comida y horarios de sueño.
  El formulario muestra un campo por ítem, y el contexto que se arme para la IA en el
  Incremento 3 llega estructurado.
- **`created_at` / `updated_at`** en CONSULTA y RECETA, con el mismo criterio que en PACIENTE
  (T4).
- **`RECETA.plan_origen_id` no se crea ahora.** Llega en el Incremento 3, con PLAN.
- **El IMC no se guarda.** Se calcula a partir del peso y la talla cuando están los dos (RF-11).
- **Valores de enum en inglés**, como todo identificador técnico (ver
  [rutas-de-api-en-ingles.md](rutas-de-api-en-ingles.md)). Viajan así en el JSON de la API, y el
  frontend muestra la etiqueta en castellano.

### Restricciones que no expresa Prisma

Son `CHECK` en el SQL de la migración. Las de condición y medición se validan también en los
schemas Zod (`patientCondition.schema.js`, `measurement.schema.js`), para responder `400` con el
campo puntual en vez de un `500` por violación de constraint.

| Constraint | Regla |
|---|---|
| `patient_condition_food_xor_group_chk` | `food_id` y `food_group_id` son excluyentes (Anexo A). |
| `patient_condition_pathology_chk` | `pathology_id` existe **si y solo si** `type = pathology`. El Anexo A anota solo un sentido ("solo si tipo es patologia"); se exige también el otro, porque una condición de tipo patología sin patología no dice nada. |
| `patient_condition_target_chk` | Una condición `pathology` no referencia alimentos. Las demás referencian exactamente uno: alimento o grupo. El catálogo de alimentos es la referencia de las condiciones del paciente (RF-24); una alergia o un rechazo sin alimento ni grupo no restringe nada. Los matices van en `observations`. |
| `measurement_detail_chk` | `detail` es obligatorio para `skinfold`, `circumference` y `other`: sin él no se sabe qué pliegue, qué circunferencia o qué parámetro se midió. |
| `measurement_value_chk` | `value > 0`. |
| `prescribed_habit_eval_chk` | La consulta que evalúa un hábito no es la que lo establece. |
| `food_group_parent_chk` | Un grupo no es su propio padre. |
| `recipe_time_chk` | `time_minutes > 0` cuando se carga. |

Las reglas que cruzan tablas no entran en un `CHECK` y quedan a cargo de los servicios de cada
módulo:

- La consulta de una medición, y la consulta que evalúa un hábito, son del mismo paciente.
- No hay ciclos en la jerarquía de grupos.
- Una receta admite al menos un tipo de comida.

Una FK compuesta escrita a mano en el SQL no sirve para esto, porque Prisma la detecta como
drift.

### Política de borrado

Lo que pertenece al paciente (condiciones, consultas, mediciones) se borra en cascada con él, y
los hábitos pautados con su consulta. Si se borra una consulta, la medición queda suelta
(`SET NULL`), porque igual pertenece al paciente. Las referencias a catálogos son `RESTRICT`:
un alimento o una recomendación se desactivan (`active = false`), no se borran.

### Correspondencia con el Anexo A

| Anexo A | Columna | Campo en Prisma / API |
|---|---|---|
| PACIENTE.antecedentes | `medical_history` | `medicalHistory` |
| PACIENTE.medicacion | `medication` | `medication` |
| PACIENTE.tratamiento_externo | `external_treatment` | `externalTreatment` |
| PACIENTE.habitos | `physical_activity`, `smoking`, `alcohol`, `meal_schedule`, `sleep_schedule` | `physicalActivity`, `smoking`, `alcohol`, `mealSchedule`, `sleepSchedule` |
| — | `cooks` | `cooks` |
| PACIENTE.gusta_cocinar | `likes_cooking` | `likesCooking` |
| PACIENTE.tiempo_cocina | `cooking_time` | `cookingTime` |
| PACIENTE.equipamiento_hogar | `home_equipment` | `homeEquipment` |
| PACIENTE.lugar_compra_habitual | `usual_shopping_place` | `usualShoppingPlace` |
| PACIENTE.comidas_fuera_hogar | `meals_outside_home` | `mealsOutsideHome` |
| PACIENTE.equipamiento_externo | `outside_equipment` | `outsideEquipment` |
| PACIENTE.ocupacion | `occupation` | `occupation` |
| PACIENTE.contexto_laboral | `work_context` | `workContext` |
| PACIENTE.patrones_ingesta | `intake_patterns` | `intakePatterns` |
| PATOLOGIA | `pathology` (`name`, `description`) | `Pathology` |
| PACIENTE_CONDICION | `patient_condition` (`registered_at` = fecha_registro, `observations`) | `PatientCondition` |
| CONSULTA | `consultation` (`motivation`, `decision_origin`, `perceived_obstacle`, `main_habit`, `observations` = observaciones_libres) | `Consultation` |
| MEDICION | `measurement` (`detail`, `value`, `unit`, `date`) | `Measurement` |
| HABITO_PAUTADO | `prescribed_habit` (`consultation_id`, `evaluation_consultation_id`, `description`, `compliance`, `evaluation_observations`) | `PrescribedHabit` |
| GRUPO_ALIMENTO | `food_group` (`parent_id` = grupo_padre_id) | `FoodGroup` |
| ALIMENTO | `food` (`active` = vigente) | `Food` |
| ALIMENTO_CATEGORIA | `food_category` | `FoodCategory` |
| RECETA | `recipe` (`preparation`, `final_note`, `cooking_method`, `time_minutes`, `purchase_place`, `origin`, `available_in_catalog`) | `Recipe` |
| RECETA_INGREDIENTE | `recipe_ingredient` (`quantity_description` = cantidad_descriptiva, `order`) | `RecipeIngredient` |
| RECETA_TIPO_COMIDA | `recipe_meal_type` | `RecipeMealType` |
| RECOMENDACION_CLINICA | `clinical_recommendation` (`title`, `text`, `active` = vigente) | `ClinicalRecommendation` |

| Enum del Anexo A | Valores |
|---|---|
| PACIENTE_CONDICION.tipo | patologia → `pathology`, alergia → `allergy`, intolerancia → `intolerance`, rechazo → `rejection`, preferencia → `preference` |
| CONSULTA.tipo | primera → `first`, seguimiento → `follow_up` |
| MEDICION.tipo | peso → `weight`, talla → `height`, pliegue → `skinfold`, circunferencia → `circumference`, colesterol → `cholesterol`, triglicéridos → `triglycerides`, glucemia → `glucose`, otro → `other` |
| HABITO_PAUTADO.cumplimiento | pendiente → `pending`, cumplido → `met`, parcial → `partial`, no_cumplido → `not_met` |
| ALIMENTO_CATEGORIA.categoria | proteína → `protein`, hidrato → `carbohydrate`, fibra → `fiber`, verdura → `vegetable`, grasa_saludable → `healthy_fat`, infusión → `infusion` |
| RECETA.metodo_coccion | horno → `oven`, hornalla → `stovetop`, microondas → `microwave`, sin_coccion → `no_cooking` |
| RECETA.origen | profesional → `professional`, generada_ia → `ai_generated` |
| RECETA_TIPO_COMIDA.tipo | desayuno → `breakfast`, almuerzo → `lunch`, merienda → `afternoon_snack`, cena → `dinner` |

## Consecuencias

- `schema.prisma` es el modelo lógico vigente y el insumo del IA3. Las tablas de arriba son la
  referencia entre el Anexo A y el esquema real.
- Los `CHECK` no aparecen en `schema.prisma`. Quien lea solo el schema no los ve: por eso los
  comentarios de cada modelo los mencionan.
- La API de pacientes no cambió en T14. `GET /patients` ya devuelve las columnas nuevas, porque
  el serializer expone todo, pero el alta y la edición las rechazan hasta la tarea del módulo
  de ficha clínica.
- Un error en el frontend o en un servicio que intente guardar una condición o medición
  inválida termina en un error de base aunque se saltee la validación Zod: la integridad no
  depende de la aplicación.
