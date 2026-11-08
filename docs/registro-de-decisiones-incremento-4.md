# Registro de decisiones del Incremento 4

Insumo del informe de avance del Incremento 4. Continúa la numeración DA de
[`registro-de-decisiones-incremento-3.md`](registro-de-decisiones-incremento-3.md)
(DA-25 a DA-38). Cada decisión tiene su fundamento completo en
[`decisiones/`](decisiones/_indice.md).

## Decisiones adoptadas en el Incremento 4

| ID | Decisión | Alternativas consideradas | Motivo y consecuencias |
|---|---|---|---|
| DA-39 | Selector de recetas y objetivo en la generación (T53). El selector manual del editor (`GET /recipes?patientId=`, RF-28) excluye, además de las recetas comprendidas en una alergia, intolerancia o rechazo, las alcanzadas por una regla `prohibited` de las patologías del paciente, con el mismo criterio de "comprendido" que la generación y el motor (`ruleScopes`, `scopeCoversFood`). Las `moderate` se siguen ofreciendo y advierten al guardar, y sin `patientId` el catálogo no se recorta. El objetivo del paciente (`patient.goal`) viaja a `nutria-ia` en el campo opcional `goal` del contrato de `POST /plan-proposals`, con el filtro de datos identificatorios y el tope de 500 caracteres de los demás textos libres. Los ingredientes de las recetas no viajan. La plantilla vigente pasa a ser `plan_proposal_v3`: la v1 más una regla que orienta las opciones y los objetivos al `goal`, sin afirmaciones clínicas. No agrega ninguna regla clínica. | Selector: dejarlo como estaba; excluir también las `moderate`. IA: no enviar nada nuevo; enviar el objetivo y los ingredientes de cada receta. | Las dos vías de agregar una receta ofrecen lo mismo: antes, a un paciente celíaco el selector le ofrecía 9 de las 17 recetas comunes con gluten, y el error aparecía recién como hallazgo al guardar, sin impedir guardar el borrador. Excluir las `moderate` sería más estricto que la generación (DA-36). El objetivo no es un dato identificatorio y no agrega un dato clínico que no viajara; antecedentes, medicación y mediciones siguen sin enviarse (RNF-07). Los ingredientes suman miles de tokens de entrada mientras RNF-14 no se cumple; con la tasa de elección de las recetas de control se decide después si conviene enviarlos. El contrato cambia en los dos repos: `nutria-ia` se despliega antes que `nutria-backend`, porque rechaza con `422` un campo no declarado. Un plan que ya tenía una receta alcanzada la conserva y el motor sigue marcando el bloqueo. La medición del 07/11/2026, con el catálogo ampliado (77 recetas comunes y 45 privadas de prueba, origen V) y la v3, reemplaza a la del 01/11/2026 como línea de base sin RAG: P95 16,7 s y 90,5 % de propuestas sin bloqueantes, y ninguna receta privada de una cuenta llegó a la otra (RNF-05). Modifica RF-28 y DA-23. Completa DA-28. Actualiza DA-12 y DA-38. Fundamento en `decisiones/selector-de-recetas-y-objetivo-en-la-generacion.md`; resultados en `mediciones-de-generacion/`. |

## Requerimientos modificados

| ID | Cambio |
|---|---|
| RF-28 | Modificado por DA-39: el filtro de recetas por condiciones del paciente excluye también las recetas con algún ingrediente alcanzado por una regla `prohibited` de sus patologías. Antes, las patologías no excluían recetas y lo derivado de ellas quedaba solo en los hallazgos del motor. Las preferencias y las reglas `moderate` siguen sin excluir. |

## Actualización de decisiones anteriores

| ID | Estado |
|---|---|
| DA-12 | Actualizada por DA-39: la plantilla vigente pasa de la v1 a `plan_proposal_v3` (la v1 más el objetivo del paciente). La adopción de la v2 sigue pendiente de revisión profesional. |
| DA-23 | Modificada por DA-39: el filtro de RF-28 aplica también las reglas `prohibited` de las patologías del paciente, la alternativa que DA-23 había descartado cuando todavía no existía el motor de reglas. Los ingredientes siguen referenciando siempre un alimento del catálogo. |
| DA-28 | Completada por DA-39: el contexto anonimizado suma el campo opcional `goal`, con el mismo filtro de datos identificatorios que los demás textos libres. |
| DA-38 | Actualizada por DA-39: la línea de base sin RAG contra la que se compara el RAG es la medición del 07/11/2026, no la del 01/11/2026, porque cambian la plantilla y el catálogo. |
