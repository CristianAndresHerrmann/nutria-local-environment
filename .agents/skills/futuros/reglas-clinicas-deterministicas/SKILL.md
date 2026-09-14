---
name: reglas-clinicas-deterministicas
description: "BORRADOR — no activar todavía. Motor de reglas clínicas deterministicas en el backend. Queda en futuros/ hasta que existan las entidades REGLA_CLINICA/HALLAZGO implementadas y la entrevista de validación con nutricionistas."
---

# Motor de reglas clínicas deterministicas (borrador, Incremento 2/3)

**Este skill no está activo.** Vive en `futuros/` para dejar fijados los
principios ya decididos, de modo que nadie —humano o asistente de IA— invente una
regla clínica antes de que corresponda. No mover a `.agents/skills/` (nivel
activo) hasta que:

1. Las entidades `REGLA_CLINICA` y `HALLAZGO` estén implementadas en el modelo de
   datos y migradas.
2. Se haya hecho la entrevista de validación con nutricionista(s) sobre qué
   reglas son correctas y cómo deben aplicarse.

## Principios ya decididos (no cambian cuando se active el skill)

- El motor de reglas vive en el **backend**, nunca en el servicio de IA.
- Las reglas clínicas son datos estructurados en PostgreSQL, no lógica embebida
  en prompts ni en texto libre.
- Ninguna sugerencia de IA se aprueba automáticamente: pasa por el motor de
  reglas y luego por revisión del nutricionista antes de quedar como plan final.
- El motor separa explícitamente información nutricional general (bibliografía,
  RAG) de restricciones clínicas obligatorias — no se mezclan como si tuvieran el
  mismo nivel de certeza.

## Al activar este skill

Reemplazar este archivo por la versión completa (con los pasos concretos para
agregar/modificar una regla), moverlo a `.agents/skills/reglas-clinicas-
deterministicas/` y quitar la marca de "borrador" de la `description`.
