# Lo que se ofrece a la IA: sin grupos amplios ni recetas con bajas, y aviso de receta fuera de su comida

**Fecha:** 08/11/2026
**Tarea relacionada:** T54 (DA-40, en [`registro-de-decisiones-incremento-4.md`](../registro-de-decisiones-incremento-4.md))

## Contexto

Después de T53 quedaban cuatro huecos entre lo que la generación le ofrece a la IA y lo que
el motor de reglas puede controlar:

- **Grupos amplios.** El catálogo permitido (DA-31) sacaba el grupo de una condición o regla
  y sus subgrupos, pero seguía ofreciendo los grupos que los contienen: "Carnes, pescados y
  huevos" con rechazo de "Pescados". En celiaquía es peor: las reglas son por alimento
  (DA-36), así que "Cereales y derivados" y todos sus subgrupos se ofrecían enteros. Si la IA
  propone "un cereal", el motor no avisa, porque un ítem de grupo más amplio que la regla no
  dispara (DA-30). Es el riesgo que DA-30 dejó aceptado para la revisión del profesional.
- **Receta en otra comida.** Nada controlaba que una receta del plan corresponda a la comida
  donde está. `nutria-ia` rechaza un código que no se envió, pero no mira la comida. El
  selector manual pide las recetas por tipo de comida, pero el guardado por API acepta
  cualquiera. En la medición de T53, la IA puso 0 de 274 recetas en otra comida.
- **Recetas con un ingrediente dado de baja.** Se ofrecían a la IA y en el selector. El
  alimento de baja no viaja en el contexto, así que la receta tenía un ingrediente sin
  código.
- **Más de 200 recetas candidatas.** El backend cortaba antes de llamar a la IA, porque el
  contrato admite hasta 200, pero con un `Error` plano que terminaba en un `500`.

Los cambios son conservadores: solo sacan opciones, no habilitan nada nuevo.

## Alternativas consideradas

**Grupos amplios:**

- **Mantener DA-30/DA-31**, que el profesional elija el alimento concreto al revisar. Sin
  cambios, pero la IA puede proponer "un cereal" a un celíaco sin ningún aviso.
- **Que el motor avise sobre un ítem de grupo que incluye algo restringido.** Cubre también
  la edición manual, pero es una regla clínica nueva que ningún requerimiento define. DA-30
  la llevó a la entrevista.
- **No ofrecerle esos grupos a la IA.** Solo saca opciones. Obliga a cambiar el contrato: hoy
  cada alimento enviado lleva el código de un grupo enviado.

**Receta fuera de su comida:**

- **Salida inválida en `nutria-ia`.** No requiere migración, pero descarta toda la generación
  (unos 15 s y su costo) por un solo ítem, y no cubre la edición manual.
- **Hallazgo bloqueante.** Es más estricto, pero una receta en otra comida no es un riesgo
  clínico, y el profesional puede querer usarla así.
- **Hallazgo de advertencia.**

**Más de 200 candidatas:** recortar la lista a 200. Se descartó porque elige en silencio qué
recetas ve la IA.

## Decisión

- **No se ofrece a la IA un grupo cuyo subárbol tenga algo alcanzado** por una condición o
  por una regla `prohibited` del paciente. Cuenta tanto el grupo de cada alimento comprendido,
  incluidos los dados de baja ("un cereal" incluye el trigo aunque esté de baja), como cada
  grupo de una condición o regla, aunque no tenga alimentos. El criterio de "comprendido" es
  el mismo de DA-23 (`conditionScopes`, `ruleScopes`, `scopeCoversFood`).
  - Los alimentos aptos de esos grupos se siguen ofreciendo, sin grupo.
  - En el contrato de `POST /plan-proposals`, `catalog.foods[].group_code` pasa a ser
    opcional. Un código informado que no está en `food_groups` sigue dando `422`. La
    plantilla no lo menciona, así que sigue siendo `plan_proposal_v3`.
  - El motor no cambia: un ítem de grupo amplio que agrega el profesional sigue sin disparar
    (DA-30).
- **Una receta en una comida que no está entre sus tipos de comida da un hallazgo de
  advertencia:** regla R-06, tipo `meal_type_mismatch`, con el ítem. Vale para la propuesta
  y para la edición manual. Requiere la migración
  `agrega_hallazgo_de_receta_fuera_de_su_comida`, que agrega el valor al enum y su rama en
  `finding_origin_chk`.
- **Una receta con algún ingrediente dado de baja no se ofrece**, ni a la IA ni en el
  selector del editor (`GET /recipes?patientId=`). La pantalla de Recetas (sin `patientId`)
  la sigue mostrando para poder corregirla.
- **Un catálogo permitido más grande que el contrato responde `422
  GENERATION_CATALOG_TOO_LARGE`**, con `details: { kind, count, limit }`, para los tres topes
  (200 recetas, 2000 alimentos, 500 grupos). No llama a la IA y no deja traza, porque no hubo
  llamada al proveedor (DA-31). El mensaje sugiere pedir menos comidas o dar de baja recetas
  propias, y el frontend lo muestra tal cual.

## Consecuencias

- **Modifica DA-31:** cambia el criterio de los grupos que se ofrecen. **Completa DA-28:**
  `group_code` opcional. **Actualiza DA-30:** el riesgo del grupo amplio queda cubierto en
  la generación. Sigue abierto en la edición manual y va a la entrevista.
- **`nutria-ia` se despliega antes que `nutria-backend`**, como en T53. Con el orden
  inverso, todo contexto con un alimento sin grupo daría `422` y fallaría la generación.
- A un paciente celíaco no se le ofrece ningún grupo de cereales. La IA propone el alimento
  concreto ("Arroz blanco", "Pan sin TACC"), o lo escribe como texto libre, que bloquea. La
  medición dirá si eso cambia la composición o los bloqueos.
- Un plan que ya tenía una receta fuera de su comida suma la advertencia en la próxima
  revalidación. Una receta con un ingrediente de baja que ya estaba en un plan se conserva.
- El `422` no cuenta como generación fallida en la traza ni en el log de fallas sin traza
  (RNF-11): no es una falla de la IA.
- **Sigue sin resolverse:** las reglas por `foodId` no alcanzan a alimentos privados ni a
  comunes nuevos, y "bloqueante" no impide guardar hasta que exista la aprobación del plan.
