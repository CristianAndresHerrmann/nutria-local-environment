# Alcance del Incremento 3: RAG y generación de recetas pasan al Incremento 4

**Fecha:** 01/11/2026
**Tarea relacionada:** T42 (DA-38)

## Contexto

El Plan de Proyecto prevé que el servicio de IA incorpore recuperación aumentada (RAG)
sobre un corpus de bibliografía nutricional, y la generación de recetas (RF-39) con el
mismo recorrido que el plan: propuesta, resolución contra el catálogo, validación y
revisión del profesional. El IA2 dejó el incremento del RAG como definición pendiente
(DA-13).

Al cierre del Incremento 3 la generación del plan funciona de punta a punta con el
proveedor elegido (DA-12), el motor de reglas (DA-30, DA-36) y la edición con
revalidación (T40). Todavía no la probó ninguna nutricionista: el feedback del staging
(T43) y la entrevista de validación son los que confirman los supuestos de la matriz de
reglas, la composición de desayuno y merienda y el catálogo (origen V).

## Alternativas consideradas

- **Incluir el RAG y RF-39 en el Incremento 3.** Cumple el Plan tal como estaba, pero
  atrasa el despliegue en staging: las nutricionistas no prueban la generación hasta que
  esté el RAG, y el RAG se diseñaría sin su feedback.
- **Desplegar en staging sin cerrar el incremento.** Las nutricionistas prueban antes,
  pero queda sin registrar qué entró y qué no, y el informe de avance no tiene un corte.
- **Cerrar el Incremento 3 sin RAG ni RF-39, desplegarlo y pasarlos al Incremento 4
  (elegida).**

## Decisión

El Incremento 3 cierra con la generación del plan sin RAG y sin generación de recetas.
Se despliega en staging para que las nutricionistas que validan el sistema lo prueben y
den feedback. El RAG (cards T45 a T51) y RF-39 pasan al Incremento 4.

## Consecuencias

- **Feedback antes del RAG.** Lo que observen las nutricionistas sobre la propuesta
  (contenido, catálogo, reglas, recetas) se registra en T43 y orienta el diseño del RAG
  (T45) y la composición del corpus (T46).
- **Línea base para comparar.** Las mediciones sin RAG del 01/11/2026
  ([`mediciones-de-generacion/`](../mediciones-de-generacion/README.md)) y las muestras
  leídas a mano ([`muestras-de-generacion/`](../muestras-de-generacion/README.md)) son la
  línea de base. Cuando entre el RAG se repiten con el mismo script y los mismos casos.
  El RAG también suma tokens de entrada y puede mover el P95 de RNF-14.
- **RF-39 sin card todavía.** Reutiliza el contrato, la resolución contra el catálogo y la
  traza del plan. El modelo deja pendiente `GENERACION_IA.tipo` y `receta_id`
  ([ajustes del Inc. 3](ajustes-al-modelo-del-anexo-a-incremento-3.md)). La card se crea
  al planificar el Incremento 4.
- **Riesgo 18 sin la mitigación del RAG.** El contenido clínico en texto libre
  (diagnóstico y recomendaciones contextuales) sigue mitigado solo por la plantilla y la
  revisión profesional (DA-28). Las recomendaciones clínicas salen del catálogo, con
  fuente.
- **DA-13 queda actualizada:** el RAG se incorpora en el Incremento 4. La biblioteca y el
  corpus siguen pendientes (T45, T46).
