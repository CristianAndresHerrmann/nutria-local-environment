# Ajustes al modelo de datos del Anexo A

**Tarea relacionada:** T4

## Contexto

El Anexo A del Informe de Avance 1 define las entidades NUTRICIONISTA y PACIENTE.
Al llevarlo a un esquema implementable para el Incremento 1 (RF-02, RF-03, RF-15,
RF-17) aparecieron diferencias con lo que piden las pantallas del Figma y las
validaciones: el nombre venía en un solo campo, el contacto también, y no había
fechas de creación ni de actualización.

## Alternativas consideradas

- **Implementar el Anexo A tal cual.** Sin desvíos que justificar. En contra: el
  listado muestra "última actualización" y no habría de dónde sacarla, y un
  `contacto` único mezcla teléfono y email, que se validan distinto.
- **Ajustar el modelo y dejar registrados los cambios.** El esquema responde a lo
  que la aplicación necesita, y el desvío queda documentado para el IA2.

## Decisión

Respecto del Anexo A:

- `nombre` se divide en **nombre** y **apellido**. Son los dos únicos campos
  obligatorios del paciente; el resto admite guardado parcial.
- `contacto` se divide en **teléfono** y **email**, para validar el email con su
  formato propio.
- Se agregan **creado_en** y **actualizado_en**. La segunda alimenta la "última
  actualización" del listado y su orden.

Además, los identificadores del esquema están en inglés y camelCase (`firstName`,
`phone`, `createdAt`), mapeados a columnas snake_case (`first_name`, `phone`,
`created_at`). Así el esquema, el JSON de la API y las rutas usan un mismo idioma
técnico (ver [rutas-de-api-en-ingles.md](rutas-de-api-en-ingles.md)). Los datos que
se cargan siguen en español.

| Anexo A | Columna | Campo en Prisma / API |
|---|---|---|
| nombre | `first_name`, `last_name` | `firstName`, `lastName` |
| contacto | `phone`, `email` | `phone`, `email` |
| — | `created_at`, `updated_at` | `createdAt`, `updatedAt` |

## Consecuencias

- La correspondencia entre el Anexo A y el esquema real no es 1:1 ni en estructura ni
  en nombres. La tabla de arriba es la referencia para el IA2.
- Las columnas clínicas y contextuales (RF-04 a RF-14) no están en este esquema.
  Entran por migración en el Incremento 2; RF-07 se va a modelar como entidad
  (PACIENTE_CONDICION), no como texto libre.
- `data_completed_at` (datos completados en) se registra solo cuando la ficha pasa
  de incompleta a completa y después no se modifica. Queda como registro auditable
  (RNF-03).
