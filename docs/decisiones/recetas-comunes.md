# Recetas básicas comunes de solo lectura, y privadas por nutricionista

**Fecha:** 29/10/2026
**Tarea relacionada:** T30

## Contexto

Hasta T30 toda receta es de un nutricionista (DA-15,
[catalogos-comunes-entre-nutricionistas.md](catalogos-comunes-entre-nutricionistas.md)). Un
nutricionista recién dado de alta no tiene ninguna, y la generación del plan (T37) solo ofrece
como candidatas las recetas que ve. T30 pide recetas básicas disponibles para todos, antes de
evaluar la generación.

## Alternativas consideradas

- **Una copia de cada receta básica por nutricionista, en el seed.** Sin migración. En contra:
  no llega a los nutricionistas que se den de alta después (DA-25) y duplica datos.
- **Receta común como los alimentos y las recomendaciones (DA-16):** `nutritionist_id`
  nullable, `NULL` es común y de solo lectura.

## Decisión

**Receta común con `recipe.nutritionist_id` nullable**, con el mismo criterio que
[recomendaciones-privadas-por-nutricionista.md](recomendaciones-privadas-por-nutricionista.md).

- Migración `recetas_comunes`: la columna pasa a nullable. La FK sigue en `RESTRICT`, explícito
  en `schema.prisma`: con la relación opcional, el default de Prisma (`SET NULL`) convertiría
  en comunes las recetas de un nutricionista borrado.
- `GET /recipes` devuelve las comunes más las propias, y los filtros por tipo de comida y por
  condiciones del paciente (RF-28, DA-23) se aplican igual a las dos. `GET /recipes/:id` muestra
  una común; una privada de otro da `404`.
- Editar o dar de baja una común da `403` (el nutricionista la ve: un `404` sería
  contradictorio). `POST /recipes` crea siempre una privada del nutricionista del token.
- La respuesta no expone `nutritionistId`: trae `own`.
- **Una receta común solo tiene ingredientes comunes**, porque si no filtraría un alimento
  privado a los demás nutricionistas. Lo valida el servicio (`upsertCommon`, sin ruta HTTP), y
  el seed carga las comunes llamándolo. Un test de los datos del seed lo controla además sin
  base.
- El seed pone al día las comunes por nombre y nunca las borra: un plan de cualquier
  nutricionista puede referenciarlas (`option_item` las restringe). Las privadas del seed se
  siguen reemplazando por nutricionista.
- El plan y la generación ven las comunes sin cambios propios: leen el mismo listado.
- En el frontend, cualquier receta, común o propia, se puede **duplicar**: se copia en el
  formulario de alta, el nutricionista la adapta y se guarda como receta privada suya. No es
  parte de la card; resuelve cómo se lee la preparación de una común y cómo se ajusta a gusto
  sin editarla.

## Consecuencias

- Todo nutricionista, también el que se da de alta después, tiene recetas candidatas desde el
  primer día.
- Las comunes se corrigen por seed hasta que el administrador las gestione (posible extensión de
  T25, que solo tendría que exponer `upsertCommon` en una ruta). Ocultar una común del catálogo
  de un nutricionista queda fuera de alcance.
- Renombrar una común en el seed deja la anterior en las bases ya cargadas, igual que con los
  alimentos.
- El contenido de las 17 recetas tiene origen V: criterios y pendientes en
  [`recetas-basicas.md`](../recetas-basicas.md).
- **Modifica DA-15**, que sigue siendo un supuesto a confirmar en la entrevista de validación.
