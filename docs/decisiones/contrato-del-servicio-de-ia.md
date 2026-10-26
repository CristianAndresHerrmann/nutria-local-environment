# Contrato del servicio de IA para generar el plan, con proveedor simulado

**Fecha:** 26/10/2026
**Tarea relacionada:** T33

## Contexto

T33 es la primera funcionalidad real de `nutria-ia`. Fija el contrato que consumen el
backend (T34 y T36) y el motor de reglas (T37), sin depender todavía del proveedor
externo, que sigue en evaluación (DA-12). El diseño parte del IA2 y del modelo de T32
(DA-27):

- El backend arma un contexto mínimo y anonimizado, y el servicio es la segunda barrera
  de RNF-07.
- La propuesta se organiza por tipo de comida, opciones e ítems.
- Cada generación registra la plantilla, el proveedor, la respuesta y el resultado
  (RF-41, RNF-11).
- Toda falla es un error recuperable (RNF-02).

Hubo que decidir cinco cosas:

- Cómo referencia la propuesta al catálogo.
- Qué textos del plan redacta la IA.
- Qué pasa con un código que el proveedor inventa.
- Con qué formato salen los errores.
- Cómo se simula el proveedor.

## Alternativas consideradas

- **Ítems por nombre o por UUID.**
  - Por nombre, el backend tendría que adivinar a qué alimento se refiere. Es el
    "parseo con regex ad hoc" que prohíbe el skill del servicio.
  - Por UUID, el prompt gasta tokens y el modelo tiende a inventar identificadores
    largos.
- **Que la IA redacte también las recomendaciones clínicas.** Aprovecharía más al
  modelo, pero:
  - El motor de reglas valida ítems estructurados (RF-43), no texto libre. Un "puede
    comer avena común" en un plan para celiaquía no generaría ningún hallazgo.
  - Las afirmaciones no tendrían fuente (riesgo 18; RF-47 la exige para las reglas).
  - Un texto clínico bien redactado invita a aceptarlo sin revisarlo.
  - Lo clínico tiende a cuantificar ("no más de 5 g de sal"), contra RF-35.
- **Que la IA elija las recomendaciones clínicas del catálogo por código.** Tendría la
  misma integridad que no generarlas, pero aporta poco: la relación entre recomendación
  y patología ya es determinística, así que el backend puede preseleccionarlas sin IA.
- **Redacción clínica con RAG y citas verificables.** Es la única forma razonable de que
  la IA escriba contenido clínico. Depende del corpus y de pgvector, que no son parte de
  esta tarea.
- **Pasar al backend un código desconocido como ítem no reconocido (DA-26).** No se
  descarta nada de la propuesta. En contra, un código inventado no trae texto original
  que conservar, y el contrato dejaría de garantizar que todo código devuelto existe.
- **Errores de proveedor con el 422 o el 500 de FastAPI.** No requiere código propio.
  En contra, el backend no podría distinguir un timeout de una respuesta mal formada
  sin parsear mensajes, y la generación no quedaría registrada.

## Decisión

### Contrato

`POST /plan-proposals`, documentado en el Swagger del servicio (y, a través del
backend, en `/docs`).

**Entrada.** Cada modelo, en todos los niveles, usa `extra="forbid"`: cualquier campo
no declarado da `422` y el proveedor no se llama. Trae lo siguiente:

- Tipos de comida pedidos.
- Condiciones del paciente, con tipo y nombre. Las observaciones no viajan, porque son
  texto libre.
- Contexto de cocina (RF-08), comidas fuera del hogar (RF-09), trabajo y horarios
  (RF-10) y hábitos (RF-06).
- Catálogo permitido, de alimentos con categorías y de grupos, más las recetas
  candidatas. Cada elemento lleva un código corto que arma el backend, único en su
  lista.

**Salida.**

- Seis textos: diagnóstico situacional, introducción, objetivos y recomendaciones
  generales, contextuales y temáticas.
- Comidas → opciones → ítems. La comida lleva la indicación de fraccionamiento (RF-36).
  Cada ítem es un alimento, un grupo, una receta (por código) o un texto libre
  marcado, que el backend guarda como ítem no reconocido (DA-26).
- No hay campos de cantidad ni de calorías (RF-35).
- Los datos de la generación: proveedor, modelo, versión de plantilla, duración, tokens
  y respuesta cruda.

**Validación de la salida.** El servicio parsea la respuesta del proveedor en modo
estricto. Después la contrasta con la solicitud:

- Un tipo de comida repetido, no pedido o faltante es salida inválida.
- Un código que no se envió para ese tipo de ítem también lo es.

### Recomendaciones clínicas

La IA no las redacta. Salen del catálogo que elige el profesional (DA-16). La plantilla
prohíbe en todos los textos:

- Las afirmaciones clínicas.
- Las cantidades y las calorías.
- Decir que el plan es seguro.

### Errores controlados

Tienen el formato del backend, `{ error: { code, message, details } }`, con
`details.generation` (y `details.errors` cuando la salida es inválida):

| Situación | Status | `code` | `ai_generation.result` |
|---|---|---|---|
| El proveedor no responde a tiempo | 504 | `PROVIDER_TIMEOUT` | `timeout` |
| El proveedor falla | 502 | `PROVIDER_ERROR` | `provider_error` |
| La respuesta no respeta el contrato | 502 | `INVALID_PROVIDER_OUTPUT` | `invalid_response` |

El `422` de entrada inválida queda con el formato estándar de FastAPI. Indica un error
del backend al armar el contexto, no una falla recuperable.

### Proveedor y plantillas

**Proveedor.** Hay una interfaz común de proveedor (riesgo 5). Hoy la única
implementación es un proveedor simulado y determinístico (riesgo 11):

- Arma una propuesta válida a partir del contexto.
- La serializa a JSON, así pasa por el mismo parseo que un proveedor real.
- Un escenario configurable provoca cada error controlado.

**Plantillas.** Hay una por versión, en un archivo. La respuesta informa la versión
usada, y un test guarda el hash de cada plantilla publicada.

## Consecuencias

- **Backend (T36).**
  - Traduce los códigos cortos a UUID al armar la solicitud y al guardar la propuesta.
  - Mapea los tres errores a uno con reintento (RNF-02).
  - Registra `ai_generation` con `details.generation`. La respuesta cruda llega también
    en `INVALID_PROVIDER_OUTPUT`, como exige `ai_generation_response_chk`.
  - Su tiempo máximo para esta llamada tiene que superar `AI_PROVIDER_TIMEOUT_SECONDS`.
- **Rechazo completo de la propuesta.** Un solo código inventado alcanza para rechazarla
  entera. Con el proveedor real, el indicador de propuestas rechazadas por la
  plantilla dice si la próxima versión del prompt tiene que reforzar esa regla.
- **Riesgo residual de contenido clínico.**
  - El diagnóstico situacional y las recomendaciones contextuales pueden traer
    contenido clínico, y el motor de reglas no lo detecta.
  - Hoy lo mitigan la plantilla y la revisión del profesional.
  - Una revisión léxica de los textos en el backend, que avise de cantidades o de
    alimentos con condición sin rechazar nada, queda como propuesta fuera de esta
    tarea.
- **Cambio de proveedor.** Agregar el real (cuando se cierre DA-12) es una implementación
  nueva de la interfaz y un valor de `AI_PROVIDER`, sin tocar el contrato.
- **Redacción clínica por la IA.** Solo se reconsidera con RAG y citas verificables.
