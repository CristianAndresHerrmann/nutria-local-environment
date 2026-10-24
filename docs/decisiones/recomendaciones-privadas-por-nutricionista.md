# Recomendaciones clínicas comunes de solo lectura, y privadas por nutricionista

**Fecha:** 24/10/2026
**Tarea relacionada:** T22

## Contexto

T14 dejó las recomendaciones clínicas como catálogo común, sin `nutritionist_id`
([catalogos-comunes-entre-nutricionistas.md](catalogos-comunes-entre-nutricionistas.md)), y T15
decidió que ningún nutricionista edita ni da de baja el catálogo común: eso es del rol
administrador, que todavía no existe ([abm-de-catalogos-comunes.md](abm-de-catalogos-comunes.md)).

T22 pide alta, edición y baja lógica de recomendaciones desde la sección Catálogos, y el
contenido real lo carga la nutricionista, no el sistema. Con el modelo de T14/T15 tal cual, no
habría quién pudiera cargarlo desde la interfaz.

## Alternativas consideradas

- **Catálogo común abierto a cualquier nutricionista.** Sin migración. En contra: lo que edita uno
  le cambia el contenido a todos, el riesgo que T15 descartó para los alimentos.
- **Un rol administrador mínimo ahora.** Fiel a T15, pero adelanta una tarea propia (rol,
  middleware, alta del administrador) en una card de prioridad media-baja.
- **Comunes de solo lectura y privadas por nutricionista**, como los alimentos. El nutricionista
  crea, edita y da de baja solo las suyas.

## Decisión

**Comunes de solo lectura y privadas por nutricionista.**

- `clinical_recommendation.nutritionist_id` nullable (FK `RESTRICT`): `NULL` es común y con valor
  es privada de ese nutricionista.
- Un nutricionista ve las comunes y las suyas. Una privada de otro no aparece en el listado, y
  pedirla o editarla por id da `404`, igual que una inexistente.
- `POST /clinical-recommendations` crea siempre una privada, con el dueño tomado del token.
- Editar o dar de baja una común da `403`: el nutricionista la ve, así que un `404` sería
  contradictorio. Las comunes se corrigen por seed hasta que exista el administrador.
- **A diferencia de los alimentos, el nutricionista sí edita y da de baja lo propio.** Ningún
  plan las referencia todavía, y una recomendación es texto, no un dato contra el que validen las
  reglas clínicas.
- La respuesta no expone `nutritionistId`; trae `own` para que el frontend ofrezca la edición
  solo en las propias.
- Sin unicidad de título: dos recomendaciones de la misma patología pueden llamarse igual.
- Las recomendaciones de hábitos de cada paciente no van en esta entidad (RF-31): son los hábitos
  pautados (T19) y el texto del plan (Incremento 3).

## Consecuencias

- **El error de un nutricionista solo lo afecta a él**, como con los alimentos privados.
- **Cuando el plan del Incremento 3 use una recomendación**, tiene que guardar una copia del
  texto, igual que con los alimentos: si no, editar la recomendación cambiaría un plan emitido.
- Pasar una privada al catálogo común, si sirve para todos, queda para la tarea del
  administrador.
- El seed carga solo textos de ejemplo identificados como ficticios (RNF-08).
- **Hay que confirmarlo en la entrevista de validación con nutricionistas.**
