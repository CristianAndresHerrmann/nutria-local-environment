# Curado del catálogo desde el panel del administrador

**Fecha:** 01/11/2026
**Tarea relacionada:** T25

## Contexto

[abm-de-catalogos-comunes.md](abm-de-catalogos-comunes.md) (DA-16) dejó la curación del
catálogo a un administrador, y [administrador-y-gestion-de-cuentas.md](administrador-y-gestion-de-cuentas.md)
(DA-25) creó ese rol, sin acceso al catálogo hasta esta tarea. Mientras tanto, un alimento
privado mal cargado no se podía corregir desde ningún lado, y el catálogo común solo cambiaba
por seed.

Abrir la edición deja tres puntos sin resolver:

- **Autoría.** "Qué alimento cargó cada nutricionista" sale de `food.nutritionist_id`. Pero
  pasar un privado al común lo pone en `NULL`, y se pierde el dato.
- **Nombres.** T15 hizo el nombre único dentro de lo que ve cada nutricionista: un común o uno
  propio, sin distinguir mayúsculas. Un común nuevo puede chocar con privados de otros
  nutricionistas, que hasta ahora no contaban.
- **Planes.** Un alimento corregido ya puede estar en planes.

## Alternativas consideradas

- **Autoría:**
  - **Solo `created_at`.** La actividad lista los privados vigentes, y uno que pasa al común
    deja de figurar como cargado por su autora.
  - **`created_by_id` aparte del dueño (elegida).** Una columna más, que se conserva al pasar
    al común.
- **Choque con privados de otros al crear o pasar un común:**
  - **Permitirlo.** La otra nutricionista vería el común y su privado con el mismo nombre en
    el buscador, hasta que el administrador diera de baja el privado.
  - **Fusionar el privado con el común**, repuntando sus condiciones, ingredientes e ítems.
    Toca datos de pacientes ajenos a la tarea del administrador (RNF-05).
  - **409, y el administrador renombra o da de baja el privado (elegida).**
- **Cambio de dueño:**
  - **En el PUT del alimento.**
  - **Con una operación aparte, `promote`, que solo pasa de privado a común (elegida).** El
    paso inverso, de común a privado, no existe.

## Decisión

- `food` suma `created_at` y `created_by_id` (FK a `nutritionist`, `RESTRICT`).
  - El alta de un privado, desde la API o desde el seed, completa el autor.
  - Los comunes del seed o del administrador quedan sin autor.
  - La migración copia `nutritionist_id` en `created_by_id` para los privados existentes. Su
    `created_at` es la fecha de la migración.
- **API bajo `/admin`, solo con `requireAdmin`.** El test de 403 cruzados la cubre sin cambios.
  - Lectura y alta y edición de grupos (con padre) y de patologías.
  - Alimentos:
    - Listado con bajas y privados de todos, con filtro `owner`, que es `common` o el id de un
      nutricionista.
    - Alta de un común.
    - Edición de cualquier alimento, con baja y reactivación (`active`).
    - `POST /admin/foods/:id/promote`, con un nombre opcional para resolver el choque.
  - Actividad del catálogo: solo los alimentos con autor, sin pacientes ni planes.
- **Jerarquía sin ciclos.** El servicio de grupos aplica `wouldCreateCycle` y responde 400 en
  `details.fields.parentId`. El frontend ni siquiera ofrece como padre al grupo ni a sus
  subgrupos.
- **Nombres (409 en `details.fields.name`):**
  - Un privado de N no repite un común ni otro privado de N.
  - Un común no repite otro común ni un privado vigente de otro nutricionista. Un privado dado
    de baja no cuenta, porque ya no se ofrece al cargar un plan. Por eso darlo de baja es una
    forma válida de resolver el choque.
  - Al pasar un privado al común, tampoco puede repetir otro privado de su dueño, aunque esté
    dado de baja.
  - En una edición, el nombre se controla solo si cambia o si el alimento se reactiva. Corregir
    las categorías de un alimento no se traba por un choque que ya existía.
  - Pasar al común un alimento que ya es común también da 409.

## Consecuencias

- **Planes.** El ítem de un plan conserva la copia del nombre y las categorías del alimento
  (DA-16), así que leer o revalidar un plan después de una corrección muestra la copia
  anterior.
  - Volver a guardar el borrador rearma la copia desde el catálogo vigente. Es lo esperable en
    un borrador, y congelarla definitivamente es tarea de la aprobación (Incremento 4).
  - Cambiar el grupo de un alimento, o el padre de un grupo, sí cambia los hallazgos en la
    próxima revalidación: las condiciones y las reglas por grupo se evalúan contra el catálogo
    vigente. Es lo correcto por seguridad clínica.
- **Un privado que pasa al común conserva su id.** Las condiciones, recetas y planes de su
  dueño no cambian, y desde ahí lo ven todos.
- **Unicidad sin respaldo en la base.** La unicidad entre comunes, y entre comunes y privados,
  sigue controlándose en el servicio: Postgres trata los `NULL` como distintos en el índice
  único, y Prisma no expresa un índice parcial sin generar drift. Con un único administrador, la
  carrera entre dos escrituras es un riesgo aceptado.
- **La FK `RESTRICT` de `created_by_id`** también impide borrar físicamente la cuenta de una
  nutricionista que cargó alimentos. La baja de cuentas ya era lógica.
- **Fuera de alcance:**
  - La baja de grupos y patologías.
  - Pasar un común a privado.
  - Fusionar alimentos duplicados.
  - El alta y la edición de recetas y recomendaciones comunes.
- Trazabilidad: RF-04 (alta de patologías), RF-24, RF-25, RNF-05. Completa DA-16 y DA-25.
