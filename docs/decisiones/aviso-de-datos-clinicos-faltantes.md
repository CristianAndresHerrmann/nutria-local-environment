# Aviso de datos clínicos faltantes calculado en el backend

**Fecha:** 16/10/2026
**Tarea relacionada:** T16

## Contexto

RF-45 pide señalar en la ficha del paciente, y más adelante en el editor del plan, que no hay
patologías, alergias, intolerancias ni medicación registradas. La advertencia no bloquea la
generación, pero en el Incremento 4 va a exigir una confirmación explícita antes de aprobar un
plan.

T16 implementa la parte de la ficha, y hay dos dificultades:

- Las patologías, alergias e intolerancias viven en `patient_condition`, y su carga llega con
  T18, que todavía no está implementada. La tabla existe desde T14 y el seed ya la llena.
- La medicación es una columna de texto del paciente (`medication`).

## Alternativas consideradas

- **El frontend avisa solo por la medicación, y T18 completa el resto.** Es lo más simple, pero
  el aviso queda mal hasta T18: aparece en un paciente con celiaquía y sin medicación.
- **`GET /patients/:id` devuelve las condiciones completas y el frontend decide.** Adelanta a
  T16 parte de la lectura de condiciones que le toca a T18. Además, la regla queda en el
  frontend, y el editor del plan del Incremento 4 (y la trazabilidad de la confirmación) la
  tendría que repetir.
- **El backend calcula un flag y lo devuelve en el detalle.** La regla queda en un solo lugar
  del backend, junto al resto de las reglas determinísticas, y se puede testear sola.

## Decisión

`GET /patients/:id` devuelve `missingClinicalData`. Es `true` cuando el paciente no tiene ninguna
condición de tipo `pathology`, `allergy` o `intolerance` y además `medication` está vacía. La
regla está en `nutria-backend/src/utils/missingClinicalData.js`.

- **Se avisa solo si faltan los cuatro datos.** El texto de RF-45 ("ausencia de patologías,
  alergias, intolerancias o medicación") admite dos lecturas. Se sigue la de la tarea: el aviso
  aparece cuando no hay ninguno de los cuatro registrados. Un paciente con una patología y sin
  medicación no muestra aviso.
- **Rechazos y preferencias no cuentan.** Hablan de gustos, no de restricciones clínicas.
- **El conteo filtra también por el nutricionista del paciente**, en la misma consulta.

## Consecuencias

- El aviso es correcto desde ya con los datos del seed. Cuando T18 habilite la carga de
  condiciones, no hay que tocarlo.
- El listado (`GET /patients`) no trae el flag, para no sumar un conteo por paciente. Si
  algún día se quiere el aviso en el listado, hace falta agregarlo ahí.
- Para el Incremento 4 queda:
  - reusar `isMissingClinicalData` en el editor del plan;
  - pedir la confirmación explícita antes de aprobar el plan;
  - registrarla en la trazabilidad.
