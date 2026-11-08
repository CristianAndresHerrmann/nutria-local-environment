# El selector de recetas aplica las reglas prohibitivas, y el objetivo del paciente viaja a la IA

**Fecha:** 07/11/2026
**Tarea relacionada:** sin card todavía (ampliación del catálogo de recetas para probar la generación)

## Contexto

Al preparar la prueba de generación con un catálogo de recetas más grande aparecieron dos
diferencias entre lo que el sistema hacía y lo que se quería comprobar:

- **Las dos vías de agregar una receta filtraban distinto.** La generación excluye las
  recetas alcanzadas por una regla `prohibited` de las patologías del paciente (DA-31,
  DA-36). El selector manual del editor (`GET /recipes?patientId=`, RF-28) solo excluía por
  alergia, intolerancia y rechazo: a un paciente celíaco le ofrecía 9 de las 17 recetas
  comunes de entonces con gluten. El error aparecía recién como hallazgo al guardar, y hoy
  un hallazgo bloqueante no impide guardar el borrador.
- **La IA no conocía el objetivo del paciente.** La lista blanca del contexto (DA-28) no lo
  incluía, aunque la propuesta redacta un texto de objetivos y se quería comprobar que las
  opciones respondan a él.

## Alternativas consideradas

**Selector manual:**

- **Dejarlo como estaba**, para que la nutricionista vea todo y decida. Sin cambios, pero
  una receta no apta se elige sin aviso previo y se puede guardar.
- **Excluir también las `moderate`.** Más estricto que la generación, que las ofrece y
  advierte (DA-36), y dejaría sin opciones a quien decida usarlas con moderación.
- **Excluir solo las `prohibited`**, con el mismo criterio que la generación.

**Datos para la IA:**

- **No enviar nada nuevo.** El objetivo seguiría sin poder evaluarse.
- **Enviar el objetivo y los ingredientes de cada receta.** La IA elegiría mejor, porque hoy
  solo ve el nombre. Pero suma miles de tokens de entrada cuando RNF-14 todavía no se cumple.
- **Enviar solo el objetivo.**

## Decisión

**El selector excluye las recetas alcanzadas por una regla `prohibited` de las patologías
del paciente.** Usa `ruleScopes` y `scopeCoversFood`, el mismo criterio de "comprendido" que
la generación y el motor (DA-23). Las `moderate` se siguen ofreciendo y advierten al guardar.
Sin `patientId`, el catálogo no se recorta. Modifica lo que la API decía para RF-28 ("las
patologías no excluyen recetas"). No agrega ninguna regla clínica.

**El objetivo (`patient.goal`) viaja a la IA, y los ingredientes no.** Es el campo `goal`,
opcional, del contrato de `POST /plan-proposals`.

- **Por qué se puede enviar:** no es un dato identificatorio, y como todo texto libre del
  contexto pasa antes por el filtro de emails, teléfonos y nombres, y por el tope de 500
  caracteres. Tampoco suma un dato clínico que no viajara: las patologías ya viajan por
  nombre. Antecedentes, medicación y mediciones siguen sin enviarse.
- **Plantilla:** `plan_proposal_v3` es la v1 más una regla que orienta las opciones y los
  objetivos al `goal`, y mantiene la prohibición de afirmaciones clínicas. Sale de la v1
  porque la adopción de la v2 sigue pendiente de revisión profesional (DA-12). Pasa a ser la
  vigente.

## Consecuencias

- El selector y la generación ofrecen lo mismo cuando el paciente cocina, y un test lo
  controla con los datos reales del seed. La generación sigue filtrando además por `cooks`:
  en el selector, esa decisión queda en la nutricionista.
- Un plan que ya tenía una receta alcanzada la conserva, y el motor sigue marcando el
  bloqueo al abrirlo.
- El contrato cambia en los dos repos. **`nutria-ia` se despliega antes que
  `nutria-backend`**: el servicio rechaza con `422` un campo no declarado, y con el orden
  inverso fallarían todas las generaciones.
- La línea de base del 01/11/2026 ya no es directamente comparable: cambian la plantilla y
  el catálogo. Hay una medición nueva en
  [`mediciones-de-generacion`](../mediciones-de-generacion/README.md), que es contra la que
  se compara el RAG.
- La elección de recetas sigue siendo por nombre. Con la tasa de elección de las recetas de
  control se decide después si conviene enviar ingredientes.
- **Sigue sin resolverse:** "bloqueante" no impide guardar hasta que exista la aprobación
  del plan, y las reglas por `foodId` no alcanzan a alimentos privados ni a comunes nuevos.
