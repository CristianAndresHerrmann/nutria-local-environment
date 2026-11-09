# Pautas de adecuación nutricional v1: pautas positivas con fuente, medibles y sin efecto en el editor

**Fecha:** 09/11/2026
**Tarea relacionada:** T57

## Contexto

Las reglas del motor (R-01 a R-06) dicen qué evitar. No había ningún criterio documentado
de qué conviene ofrecer. Sin él no se puede decir si un plan es adecuado, ni medir si el
RAG lo mejora frente a la línea de base sin RAG del 07/11/2026 (DA-39). T58 necesita una
referencia de evaluación, T45 y T61 necesitan saber qué tiene que recuperar el RAG, y T65
necesita las preguntas para la entrevista.

Las pautas se sacaron del corpus bibliográfico del proyecto (`datos-rag/`), con la cita
literal verificada contra el texto de su página. Al bajarlas al plan quedaban varios puntos
sin definir:

- Qué fuentes usar.
- Qué hacer con las pautas que el plan no puede expresar, porque no tiene cantidades ni
  días.
- Qué efecto tienen en el editor.
- Qué objetivos del paciente tienen pautas.
- Qué origen llevan.

## Alternativas consideradas

- **Pautas como reglas del motor, con advertencia en el editor.** Darían visibilidad
  inmediata, pero convertirían recomendaciones ("se pueden incluir", "se aconseja") en
  hallazgos. Sumarían advertencias a cada plan e inventarían reglas clínicas que ninguna
  fuente plantea como restricción (riesgo 15).
- **Pautas en la plantilla del prompt.** Llegarían a la IA sin esperar al RAG, pero
  duplicarían lo que el RAG tiene que recuperar del corpus. Además, la evaluación no
  podría distinguir el efecto del RAG del de la plantilla.
- **Todas las pautas por opción.** Es lo más fácil de medir, pero la GAPA plantea casi
  todo para el día ("a diario", "por día"), no para cada comida. Exigir pescado en cada
  opción contradice la fuente.
- **Sumar fuentes no argentinas (ADA/EASD 2026, OMS).** Son más recientes en diabetes,
  pero el corpus pide no mezclar jurisdicciones, y las GPC del Ministerio de Salud siguen
  vigentes.
- **Solo las GPC de patología, sin la GAPA.** Los casos C4 a C7 de T41 no tienen
  patología, y quedarían sin ninguna referencia.

## Decisión

- **Las pautas viven en `docs/pautas-de-adecuacion.md`**, separadas de las reglas. No
  generan hallazgos, no bloquean ni advierten, y no se copian al prompt. Son la referencia
  de evaluación de T58. El RAG lleva las fuentes a la IA (DA-13).
- **Fuentes:** las GPC y guías del Ministerio de Salud de la Nación y de ANMAT que ya
  están en el corpus: GAPA y su Manual de aplicación, GPC DM2 2019, GPC HTA breve 2026,
  GPC de enfermedad celíaca 2016, guía ANMAT de menú libre de gluten 2023 y GPC de
  obesidad 2023 (versión breve).
- **Origen S en cada pauta**, con obra y página. **Origen V en los criterios de medición**
  (CM-01 a CM-09): cómo se traduce una pauta al catálogo y cuándo se aplica.
- **Dos niveles de medición.** Una pauta que la fuente ata a una comida se mide por
  opción. Una pauta diaria se mide por plan: cumple si al menos una opción la cumple. Lo
  que necesita cantidades, días o atributos que el catálogo no tiene queda como "no
  medible", con su cita, y no se evalúa.
- **Objetivos:** solo "bajar de peso" tiene pautas, de la GPC de obesidad 2023 en su
  versión breve. Los objetivos sin guía oficial (por ejemplo, aumentar masa muscular)
  quedan V y sin completar.
- **Atributos que el catálogo no tiene** (integral, descremado, magro) se aproximan con
  la categoría `fiber` y el nombre del alimento. Con origen V, y sin cambio de esquema.

## Consecuencias

- **T58 tiene un contrato.** Cada pauta tiene ID, nivel, comidas e indicador sobre
  grupos, categorías o alimentos del catálogo. Las categorías y el "comprendido en" son
  los mismos del motor (`itemCategories.js`, `scopeCoversFood`).
- **Sin cambios de código, de esquema ni de seed.** El editor no cambia.
- **Solo DM2 y OBE están en el corpus inicial.** Las pautas de GAPA, MAN, HTA, CEL y ALG
  se pueden medir, pero el RAG no las va a recuperar hasta que T46 habilite esas fuentes.
  Si no se habilitan, el RAG no puede mejorar esos indicadores.
- **La medición por plan es débil:** una sola opción con pescado cumple la pauta. Es lo
  único medible sin cantidades ni días, y queda como pregunta para T65.
- **Ningún caso de T41 tiene el objetivo "bajar de peso".** Las pautas OBE no se van a
  ejercitar en la evaluación hasta que haya un caso con ese objetivo. El objetivo es texto
  libre, y no hay un mapeo automático a las pautas (CM-08).
- **Choques con reglas, que se llevan a T65:**
  - La alternativa de OBE p. 33 (vegetal con proteína o con hidrato) frente a R-03.
  - La fruta del plato de DM2 frente a DA-29.
  - Los cereales con gluten de la GAPA en un celíaco (CM-03).
- **Queda pendiente:**
  - Leer a mano las pp. 29, 31 y 33 de la versión breve de OBE, que son imágenes sin
    texto.
  - Incorporar la versión extensa de OBE (T46).
- **El origen lo confirma una persona.** Cuando la entrevista (T65) confirme una pauta o
  un criterio, pasa a E y se actualizan el documento y esta ficha.
- Relacionada con DA-13 (RAG), DA-29 (composición por comida) y DA-36 (reglas por
  patología). No modifica ninguna.
