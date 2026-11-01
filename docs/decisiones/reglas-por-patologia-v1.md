# Reglas por patología v1: matriz con fuente bibliográfica, sin entrevista

**Fecha:** 01/11/2026
**Tarea relacionada:** T35

## Contexto

T35 carga la matriz de reglas clínicas (RF-47) en `clinical_rule` y le suma al motor la
regla por patología (RF-43). La matriz iba a salir de la entrevista de validación con
nutricionistas, pero la entrevista no se hizo. Se aplica la contingencia del riesgo 2 del
Plan: reglas básicas para celiaquía, diabetes e hipertensión, con fuente bibliográfica,
que quedan pendientes de revisión.

Las fuentes salen del corpus bibliográfico del proyecto (`datos-rag/`). Cada cita se
verificó contra el texto extraído de su página:

| Clave | Obra | Páginas |
|---|---|---|
| CEL | Ministerio de Salud de la Nación. *Guía de práctica clínica sobre diagnóstico y tratamiento de la enfermedad celíaca en el primer nivel de atención*. 2016. | 35, 43 |
| ALG | ANMAT / INAL. *Guía de recomendaciones para un menú libre de gluten seguro*, 2.ª ed. 2023. | 15 |
| DM2 | Ministerio de Salud de la Nación. *Guía de Práctica Clínica Nacional sobre Prevención, Diagnóstico y Tratamiento de la Diabetes Mellitus Tipo 2*. 2019. | 245, 251, 253 |
| HTA | Ministerio de Salud de la Nación. *Guía de Práctica Clínica Nacional. Prevención, Diagnóstico y Tratamiento de la Hipertensión Arterial — versión breve*. 2026. | 8, 14 |
| GAPA | Ministerio de Salud de la Nación. *Guías alimentarias para la población argentina*. 2016. | 26 |

Al bajarlo a datos quedaban cinco puntos sin definir:

- El origen de las reglas.
- Qué severidad lleva cada una.
- Qué hallazgo produce cada severidad (pendiente desde DA-30).
- Cuánto abarca diabetes, porque la GPC DM2 dice "evitar" sobre una lista amplia de
  ultraprocesados.
- Dónde se guarda el origen.

## Alternativas consideradas

- **Pasar la card al Incremento 4.** El motor seguiría sin validar las patologías de los
  casos C1–C3, y la generación seguiría ofreciendo a la IA la avena común en celiaquía.
- **Origen V, como decía la card.** El catálogo define V como "sin fuente clínica propia",
  y estas reglas tienen fuente con páginas.
- **Todas las reglas prohibitivas y bloqueantes.** Es más cauto, pero el azúcar o la sal
  bloquearían cualquier plan. Además, contradice fuentes que dicen "limitar" o "reducir",
  no excluir.
- **Cargar solo las exclusiones explícitas.** Se pierden todas las reglas de diabetes y
  casi todas las de hipertensión.
- **Diabetes con la lista completa de la GPC DM2 (pp. 251-252):** manteca, margarina,
  fiambres, aderezos, tapas, caldo. Es más fiel a la guía, pero son muchas advertencias
  por plan. Varias no son específicas de la glucemia, sino de las grasas saturadas o del
  sodio.
- **Una columna de origen en `clinical_rule`.** El sistema no usa el origen para decidir
  nada. El plan siempre lo valida el profesional.

## Decisión

- **Origen S** en toda la matriz. Se implementa como definitiva y se revisa con una
  nutricionista: cuando la revisión la confirme pasa a E, y si la corrige se cambia la
  regla. El origen vive solo en `docs/reglas-clinicas.md` y en el comentario del archivo
  de datos del seed. **No hay migración.**
- **La severidad sale de la redacción de la fuente:**
  - "Exclusión", "descartar" o "no usar": `prohibited`. El hallazgo es `blocking`, y el
    alimento no se le ofrece a la IA (DA-31).
  - "Evitar", "limitar", "reducir" o "no se recomienda": `moderate`. El hallazgo es
    `warning` y no impide aprobar (RF-44). El alimento se le sigue ofreciendo a la IA.
- **Alcance de cada patología:**
  - **Celiaquía** (CEL pp. 35, 43; ALG p. 15). Los 27 alimentos del catálogo que
    contienen trigo, avena, cebada o centeno son `prohibited`. La avena sin TACC es
    `moderate` ("en Argentina no se recomienda que los pacientes consuman alimentos con
    avena"). Las reglas van por alimento, porque la taxonomía es neutra y las variantes
    sin TACC comparten subgrupo con las que tienen gluten (DA-21).
  - **Diabetes** (DM2 pp. 245, 251, 253). Solo azúcares y dulces, todos `moderate`:
    azúcar, azúcar mascabo, mermelada, dulce de leche, galletitas dulces simples y
    bizcochos de grasa.
  - **Hipertensión** (HTA pp. 8, 14; GAPA p. 26):
    - Caldo en cubo: `prohibited` ("No usar … caldos deshidratados como condimentos").
    - Sal fina: `moderate`.
    - El grupo Fiambres y embutidos: `moderate`.
- **Qué alimento del catálogo "contiene" trigo, avena, cebada o centeno** es un criterio
  del desarrollador, con origen V, igual que el catálogo (DA-32).
- **El motor** evalúa las reglas vigentes de las patologías del paciente con el mismo
  criterio de "comprendido en" que la condición directa (DA-23, DA-30): alimento,
  ingrediente de receta, o ítem de grupo dentro del grupo de la regla. Produce un
  hallazgo por cada par de ítem y regla, y el mensaje nombra la patología y la fuente.
- **Las reglas se versionan en el seed**, no en una migración. Viven en
  `scripts/seed-data/clinical-rules.js` y se buscan por patología y alimento o grupo. El
  seed pone al día las existentes y da de baja (`active = false`) las que salen de la
  matriz, porque los hallazgos las referencian.

## Consecuencias

- Los casos C1–C3 de T41 tienen resultado esperado, y un test lo controla con los datos
  reales del seed.
- **Con la matriz cargada, la generación excluye del catálogo enviado a la IA los
  alimentos con gluten y el caldo en cubo.** La medición de T41 tiene que repetirse antes
  de comparar el porcentaje sin bloqueantes.
- La generación toma las reglas prohibitivas de las mismas entradas del motor. Ya no hace
  una consulta aparte.
- **Riesgos aceptados**, que pasan a "Pendientes para la entrevista":
  - Un ítem de grupo amplio ("un pan") no avisa por celiaquía, porque las reglas son por
    alimento (DA-30).
  - Los productos industrializados que exigen el listado ALG (fiambres, condimentos) no
    se restringen, porque el catálogo no tiene marcas.
  - El azúcar como tratamiento de una hipoglucemia (DM2 p. 150) no se contradice: el
    azúcar solo advierte.
- **Quedan fuera** las conservas y la salsa de soja para hipertensión (ninguna fuente de
  HTA del corpus las nombra), y la enfermedad renal y la obesidad (no están en el
  catálogo de patologías, y sus guías no nombran alimentos).
- Si se agrega un ABM de reglas, va a tener que convivir con el seed. Hoy el seed da de
  baja cualquier regla vigente que no esté en la matriz.
- Completa DA-30, que dejaba la severidad del hallazgo de patología para T35. Completa
  DA-31.
