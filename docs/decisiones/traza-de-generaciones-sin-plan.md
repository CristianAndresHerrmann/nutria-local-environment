# Generación del plan: traza sin plan ante una falla, catálogo permitido, errores y datos en texto libre

**Fecha:** 26/10/2026
**Tarea relacionada:** T37

## Contexto

T37 une el contrato del servicio de IA (T33, DA-28), el modelo del plan (T32, DA-27) y
el motor de reglas (T34, DA-30) en la generación asistida del borrador. Hubo que
resolver cuatro cosas:

- **Qué se escribe cuando falla la IA.** Dos fuentes chocaban:
  - La card de T37 pide que una falla del servicio de IA responda `503` sin escribir, y
    que nunca quede un plan a medias (RNF-02).
  - DA-27 dejó `ai_generation.plan_id` obligatorio porque "el borrador se crea antes de
    invocar a la IA, así una generación fallida también queda registrada". DA-28 pide
    registrar `details.generation` también en los errores.
  - La estrategia de pruebas pide para RNF-11 "exactamente una traza por generación". El
    indicador de propuestas rechazadas por plantilla (DA-28) también necesita las
    fallidas.
- **Cómo se aplica RF-38** ("incluir u omitir recetas según la preferencia y el tiempo de
  cocina del paciente"). `cooking_time` es texto libre ("30 minutos por día"), así que no
  se puede comparar con el `time_minutes` de una receta sin parsear texto.
- **Con qué código sale una salida inválida.** La card dice "error controlado", y DA-28
  dice que el backend mapea los tres errores del servicio "a uno con reintento".
- **Qué reglas de patología excluyen del catálogo permitido.** La card pide excluir lo
  comprendido "en reglas de patología cuando existan". La severidad `moderate` (a
  moderar) no prohíbe.
- **Datos identificatorios en los textos libres.** La lista blanca del contexto deja
  afuera los campos identificatorios, pero no impide que el profesional escriba un
  nombre, un teléfono o un email dentro de `workContext`, `mealSchedule` y los demás
  textos libres que sí viajan (RNF-07).

## Alternativas consideradas

- **Falla de la IA:**
  - *No escribir nada, solo el log.* Cumple la card al pie de la letra y no requiere
    migración. En contra, pierde la traza de las fallidas (RNF-11) y el indicador de
    DA-28.
  - *Mantener DA-27: borrador vacío y generación fallida colgada de él.* No requiere
    migración. En contra, deja un plan vacío por cada falla, que es lo que la card prohíbe.
  - *Traza sin plan.* Solo se escribe la fila de `ai_generation`, colgada del paciente.
    Requiere una migración.
- **RF-38:**
  - *Un campo estructurado de minutos en el paciente* para filtrar por `time_minutes`.
    Es determinístico, pero suma una migración y un cambio de formulario fuera del
    alcance.
  - *Sin filtro*, todo RF-38 al modelo. Es lo más simple, pero ofrece recetas al horno a
    un paciente que no cocina.
  - *Filtro duro mínimo*: solo cuando se sabe que el paciente no cocina.
- **Salida inválida:** un `502` con código propio, distinto del servicio caído. El
  frontend tendría dos casos para una misma acción (reintentar).
- **Reglas:** excluir también las `moderate`, o no filtrar por reglas hasta T35.
- **Texto libre:** omitir el campo entero si contiene un dato identificatorio (pierde
  contexto útil), o dejarlo como riesgo residual sin filtro.

## Decisión

- **Una generación fallida deja su traza sin plan.** El borrador se crea solo cuando la
  generación salió bien, en la misma transacción que sus ítems, sus hallazgos y su traza.
  Migración `agrega_traza_de_generaciones_sin_plan`:
  - `ai_generation.plan_id` pasa a nullable.
  - Se agregan `patient_id` y `nutritionist_id`, con una FK compuesta a
    `patient(id, nutritionist_id)`, como la del plan.
  - El `CHECK` `ai_generation_plan_chk` exige plan si y solo si `result = 'success'`.

  La traza se escribe cuando el servicio informó la llamada al proveedor
  (`provider_timeout`, `provider_error`, `invalid_output`). Si el servicio de IA está caído
  o no responde dentro del tope del backend, no hay proveedor, modelo ni plantilla con qué
  llenarla, y la falla queda solo en el log.
- **RF-38:** si el paciente no cocina (`cooks = false`), solo se ofrecen recetas
  `no_cooking`. El tiempo y el gusto por cocinar los evalúa el modelo, que recibe
  `time_minutes` de cada receta y el contexto de cocina. La preferencia también la evalúa
  el modelo: las condiciones de tipo preferencia ya viajan en el contexto, y el contrato
  no tiene cómo marcar una receta como preferida.
- **Errores:** toda falla de la generación responde `503 IA_UNAVAILABLE`, con `reason`:
  `unreachable`, `timeout`, `provider_timeout`, `provider_error`, `invalid_output`,
  `bad_status` o `bad_body`. Un `422` del servicio indica un contexto mal armado por el
  backend, no una falla recuperable, y responde `500`.
- **Reglas de patología:** solo las vigentes con severidad `prohibited` excluyen del
  catálogo enviado, con el mismo criterio de alcance que las condiciones (DA-23). Las
  `moderate` se envían, y el hallazgo lo decide el motor (T35).
- **Texto libre:** antes de enviar, en cada texto libre del contexto se reemplazan por
  `[dato omitido]` los emails, los teléfonos (7 o más dígitos) y cada palabra de 3 o más
  letras del nombre y apellido del paciente y de su responsable, como palabra completa y
  sin distinguir mayúsculas ni acentos.
- **Fallas sin traza:** cuando no hay datos del proveedor, la falla deja en el log una
  línea JSON (`ai_generation_untraced`, con `reason`, el id del paciente y la duración),
  para contarla junto con las trazas.

## Consecuencias

- **Modifica DA-27.** El plan ya no se crea antes de invocar a la IA. Cada generación
  tiene exactamente una traza, salvo cuando el servicio no responde. Una fallida queda
  aislada por su paciente y se borra con él.
- La base no puede verificar que el plan de una generación sea de su mismo paciente
  (cruza tablas): lo garantiza el servicio, que escribe ambos en la misma transacción.
- La generación tarda lo que tarda el proveedor. La llamada queda fuera de la
  transacción, con un tope propio (`IA_GENERATION_TIMEOUT_MS`, 30 s por defecto), mayor
  que el del proveedor en el servicio (25 s). Así un timeout del proveedor llega como
  error controlado, con su traza.
- **RF-38 queda parcial en lo determinístico.** Una receta larga puede llegar a un
  paciente con poco tiempo, si el modelo no la descarta. El profesional la reemplaza al
  revisar el plan. Si la entrevista de validación lo pide, el campo estructurado es una
  migración aditiva.
- El frontend maneja un solo error de la generación, con reintento. El `reason` sirve
  para el mensaje y para el diagnóstico.
- **El filtro de texto libre es determinístico, no un reconocedor de entidades:** no
  detecta nombres de terceros ni datos escritos de otra forma. Un apellido que coincide
  con una palabra común se reemplaza también en otros usos. Queda como riesgo residual,
  mitigado por la revisión del profesional al cargar la ficha y por el registro del
  contexto enviado en cada traza.
- `clinicalRule.repository` nace en T37 con una sola consulta; T35 lo amplía para el
  motor.
