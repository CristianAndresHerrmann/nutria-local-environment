# Catálogos curados por un administrador; alimentos privados por nutricionista

**Fecha:** 15/10/2026
**Tarea relacionada:** T15

## Contexto

T14 decidió que alimentos, grupos de alimentos y patologías son catálogos comunes a todos los
nutricionistas ([catalogos-comunes-entre-nutricionistas.md](catalogos-comunes-entre-nutricionistas.md)).
Esa decisión dejó una condición: el ABM no podía quedar abierto a cualquier usuario sin decidir
quién lo administra.

T15 abre la carga manual (RF-24, "base mínima controlada de carga manual"), y hay dos riesgos:

- Que un nutricionista modifique o dé de baja algo que usan otros.
- Que se modifique un alimento que ya forma parte de un plan alimentario (Incremento 3). En ese
  caso, el plan emitido cambia sin vuelta atrás.

## Alternativas consideradas

- **Escritura abierta a todo nutricionista.** Cualquiera pisa lo que cargó otro.
- **Catálogo compartido con dueño por ítem.** Cualquiera da de alta para todos, pero cada ítem
  lo edita solo su autor. Evita que un nutricionista pise lo de otro, pero:
  - un autor igual puede modificar un alimento que ya está en un plan;
  - el error de uno le llega a todos.
- **Congelar los ítems en uso**, permitiendo editar solo lo que nada referencia. Corregir algo
  en uso obliga a darlo de baja y recargarlo, lo que choca con la unicidad del nombre.
- **Curado por un administrador, con alimentos privados por nutricionista.** Ningún
  nutricionista edita el catálogo. Lo que agrega cada uno queda para él, y las correcciones las
  hace un administrador.

## Decisión

**Curado por un administrador, con alimentos privados por nutricionista.**

- **Grupos y patologías** son siempre comunes. Para los nutricionistas son de solo lectura.
- **Alimentos:** `food.nutritionist_id` nullable. `NULL` es común; con valor es privado de ese
  nutricionista.
  - Un nutricionista ve los comunes y los suyos. Un privado de otro no aparece en el listado, y
    pedirlo por id da `404`.
  - El nutricionista solo da de alta: lo que carga queda privado, con el dueño tomado del token.
    No edita ni da de baja nada, ni siquiera lo propio.
- **El administrador** es un rol aparte, que llega en una tarea propia. Crea, modifica y da de
  baja cualquier ítem del catálogo, común o privado.
  - Si un nutricionista quiere corregir un alimento, se lo pide al administrador.
  - Hasta que exista el rol, el catálogo común se corrige por seed o por migración.
- **El administrador solo tiene funciones operativas.** No accede a pacientes, consultas,
  fichas ni planes; la gestión de pacientes es de cada nutricionista (RNF-05). De la actividad de
  los nutricionistas, puede ver la del catálogo: qué alimento cargó cada uno y cuándo.
- **Baja lógica, sin borrado físico.** Las FK `RESTRICT` de condiciones e ingredientes impiden
  borrar un alimento en uso también desde la base.
- **Jerarquía sin ciclos.** Un `CHECK` no puede impedir un ciclo de más de un nivel, así que la
  regla está en `src/utils/foodGroupTree.js`, lista para el alta y la edición de grupos del
  administrador. El `CHECK food_group_parent_chk` de T14 sigue cubriendo el caso de un grupo que
  es su propio padre.

## Consecuencias

- **El error de un nutricionista solo lo afecta a él**, porque lo que carga es privado.
- **El motor de reglas del Incremento 3** valida contra los comunes más los privados de ese
  nutricionista. Un alimento privado siempre está en un grupo común, así que las reglas por grupo
  lo alcanzan.
- **Una corrección del administrador sobre un alimento en uso sigue cambiando lo que lo
  referencia.** Por eso, el ítem del plan del Incremento 3 tiene que guardar una copia del
  alimento (nombre y categorías) además de su `food_id`: un plan emitido no cambia por una
  corrección posterior del catálogo.
- **Puede haber privados repetidos entre nutricionistas** (cada uno su "Kéfir"). Pasar un
  privado al catálogo común, si sirve para todos, queda para la tarea del administrador.
- **Hasta que exista el administrador**, un error en un alimento privado no se puede corregir
  desde la interfaz.
- **La FK `RESTRICT` de `food.nutritionist_id`** impide borrar un nutricionista que cargó
  alimentos.
- **Hay que confirmarlo en la entrevista de validación con nutricionistas.**
