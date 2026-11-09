# Orientación del plan generado: grupos con alimentos a moderar, recomendaciones clínicas, texto de uso y plantilla v4

**Fecha:** 09/11/2026
**Tarea relacionada:** T57 (DA-43, en [`registro-de-decisiones-incremento-4.md`](../registro-de-decisiones-incremento-4.md))

## Contexto

La revisión nutricional de la medición sin RAG del 09/11/2026 (`mediciones-de-generacion/`)
encontró cuatro problemas que no dependen del RAG:

- **Una falla por ítem de grupo.** A un paciente hipertenso (C3), la IA le propuso "Carnes,
  pescados y huevos" como ítem de grupo. El grupo incluye Fiambres y embutidos, que son
  `moderate` en hipertensión, y el motor no advirtió nada: un ítem de grupo más amplio que
  la regla no dispara (DA-30). DA-40 sacó del catálogo enviado los grupos que contienen algo
  **prohibido**, pero no los que contienen algo **a moderar**.
- **Intolerancia sin alternativa.** Al paciente con intolerancia a la leche (C5), la IA no
  le ofreció leche deslactosada ni ningún otro lácteo, y el plan quedó sin una fuente clara
  de calcio.
- **Sodio en hipertensión.** Ningún texto del plan lo menciona. La plantilla le prohíbe a
  la IA las indicaciones por patología (regla 6), y las recomendaciones clínicas del plan
  quedaban vacías (DA-28): el profesional las tenía que escribir a mano.
- **Porciones y opciones.** El plan no tiene cantidades (RF-35). Nada aclaraba que las
  opciones de una comida son alternativas (no tres platos ni tres tazas), ni cómo armar el
  plato. Tampoco se sugería fruta cuando el almuerzo o la cena no la tenían.

La avena sin TACC queda `moderate` (DA-36). Se ofrece y advierte, y la decide el
profesional según el paciente. Los pacientes pediátricos (C7 tiene 12 años) quedan para
una tarea de investigación de fuentes.

## Alternativas consideradas

- **Grupo amplio con un alimento a moderar.**
  - Que el motor advierta un ítem de grupo que contiene algo a moderar. Descartada: es una
    regla nueva sin fuente (V), y sigue sin cubrir el ítem que agrega el profesional.
  - No ofrecer esos grupos a la IA, como DA-40 con los prohibidos (elegida). Solo saca
    opciones: el alimento moderado sigue disponible suelto, y advierte.
- **Sodio.**
  - Que la IA lo redacte. Descartada: choca con la regla 6 de la plantilla, que existe
    porque el motor no valida texto libre (riesgo 18).
  - Copiar al plan las recomendaciones del catálogo de las patologías del paciente
    (elegida). Es lo que DA-28 dejó previsto ("las recomendaciones clínicas salen del
    catálogo, con fuente"). El texto lo carga la nutricionista, o el seed con su cita.
- **Texto de uso y método del plato.**
  - Que lo escriba la IA. Descartada: puede omitirlo o redactarlo distinto en cada
    generación.
  - Un texto fijo del backend (elegida). Aparece siempre, y no nombra alimentos, para no
    contradecir una condición.
- **Fruta de postre.**
  - Texto fijo. Descartada: no ve las condiciones del paciente, y sugeriría fruta ante una
    alergia a las frutas.
  - Una regla de la plantilla (elegida). La IA conoce las condiciones y las opciones que
    armó.
- **Intolerancia.**
  - Una regla de la plantilla, solo para intolerancias (elegida). Ante una alergia a la
    leche, la deslactosada sigue siendo leche, y el catálogo permitido no la excluye cuando
    la condición está cargada sobre el alimento "Leche".

## Decisión

1. **Catálogo enviado a la IA:** tampoco incluye los grupos cuyo subárbol contiene un
   alimento alcanzado por una regla `moderate` del paciente. Los alimentos, las recetas y
   los subgrupos aptos se siguen enviando. Modifica DA-40.
2. **Recomendaciones clínicas:**
   - Al generar, se copian al plan las vigentes de las patologías del paciente, comunes y
     propias del nutricionista, como texto, con su título.
   - Si no entran todas en el tope de los textos del plan, se copian enteras las que caben y
     el texto avisa cuántas quedaron afuera.
   - Completa DA-28.
3. **Recomendación común "Alimentación reducida en sodio" para Hipertensión,** cargada por
   el seed:
   - Texto: las indicaciones de la GPC HTA, versión breve (2026), p. 14, citadas
     textualmente y con la fuente. Origen S.
   - Diabetes y Celiaquía siguen con su texto de ejemplo ficticio.
   - El seed da de baja las recomendaciones comunes que salen de su lista.
4. **Texto de uso del plan,** al principio de las recomendaciones generales: las opciones son
   alternativas (una por comida), y almuerzo y cena se arman con el método del plato:
   - El método del plato tiene origen S: GPC DM2, p. 248.
   - Que las opciones sean alternativas tiene origen P, por la estructura del plan.
5. **Plantilla `plan_proposal_v4`:** la v3 con tres reglas:
   - Ante una intolerancia, incluir la variante del catálogo sin el componente que la
     motiva.
   - Sugerir fruta fresca de postre si el almuerzo o la cena no la tienen, salvo que una
     condición alcance a las frutas.
   - No repetir el texto de uso.
   - Actualiza DA-12.
6. **Los textos del plan admiten hasta 10000 caracteres** (antes 4000), para que un borrador
   generado se pueda volver a guardar tal cual.
7. **Premisa del proyecto: los productos envasados del catálogo** (panes, quesos, conservas,
   aceitunas) **se consideran versiones sin el sello "exceso en sodio"**. Por eso no
   llevan regla en hipertensión. La recomendación de Hipertensión le pide al paciente que
   elija esas versiones, con la frase de la GPC HTA, p. 14. Las reglas siguen cubriendo lo
   que es alto en sodio por naturaleza: sal fina, caldo en cubo, salsa de soja, fiambres y
   embutidos.

## Consecuencias

- **Contrato de `nutria-ia`:** no cambia. Las reglas nuevas usan campos de salida que ya
  existían, así que no hay un orden de despliegue entre los repos.
- **Pautas en el prompt (DA-41):** la v4 incluye indicaciones de presentación (postre) y de
  elección de alimentos (variante sin lactosa), no la tabla de pautas. Lo demás lo sigue
  llevando el RAG desde el corpus, y se mide con las pautas.
- **Planes ya guardados:** no cambian. Las recomendaciones y el texto de uso aparecen en los
  planes generados a partir de ahora.
- **Staging:** necesita volver a correr el seed. Si no, la recomendación de Hipertensión
  sigue siendo el ejemplo ficticio.
- **Línea de base:** cambian la plantilla y el catálogo enviado, así que se volvió a medir
  con la v4 el 09/11/2026 (`mediciones-de-generacion/`):
  - 20 de 21 propuestas sin bloqueantes (95,2 %). El único bloqueo es texto libre de comida
    comprada.
  - P95 de 15,5 s.
  - 15 de 21 sin ningún hallazgo.
  - Los 21 planes traen el texto de uso. Las clínicas de C3 traen la recomendación de
    sodio.
  - C5 recibió leche deslactosada en las 3 corridas.
- **Pruebas:**
  - `allowedCatalog.test.js` y `clinical-cases.rules.test.js`: grupos con un alimento a
    moderar, con el catálogo real del seed.
  - `planTexts.test.js`: texto de uso y tope de las clínicas.
  - `planGeneration.service.test.js` y `planGeneration.service.int.test.js`: recomendaciones
    copiadas, sin las de otra cuenta ni las dadas de baja.
  - `seed.int.test.js`: la baja de una recomendación común que sale de la lista.
  - `test_prompts.py`: la v4.
- **Queda para T65:**
  - El texto de sodio.
  - La avena sin TACC.
  - La sal con bajo contenido de sodio en hipertensión.
