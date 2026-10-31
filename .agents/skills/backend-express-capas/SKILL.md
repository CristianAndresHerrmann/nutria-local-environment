---
name: backend-express-capas
description: Cómo armar un endpoint completo en nutria-backend siguiendo el esquema de capas del proyecto. Usar al agregar o modificar un endpoint del backend.
---

# Endpoint completo — backend (Node.js/Express + Prisma)

Este skill opera dentro de `nutria-backend/`. Todo endpoint nuevo sigue el mismo
orden de capas, sin saltarse ninguna:

1. **Schema Zod** — valida el body/query/params de la request. Es la única fuente
   de verdad de qué forma tienen los datos de entrada. Una ruta sin parámetros de
   query usa `emptyQuerySchema` (`src/schemas/common.schema.js`).
2. **Repositorio** — acceso a datos vía Prisma. No contiene lógica de negocio.
3. **Servicio** — lógica de negocio. Es la capa que aplica reglas (por ejemplo,
   que un paciente menor de edad requiera responsable).
4. **Controlador** — orquesta: lee los datos ya validados de `req.validated`,
   llama al servicio, arma la respuesta. No valida por su cuenta ni accede a Prisma
   directamente. Los errores se lanzan como `AppError` y se pasan a `next`.
5. **Ruta** — registra el endpoint y engancha la validación con
   `validate({ params, query, body })` (`src/middlewares/validate.js`) entre
   el middleware de rol y el controlador. Toda ruta protegida usa
   `requireNutritionist` o `requireAdmin` (`src/middlewares/auth.js`), nunca
   `requireAuth` suelto: así el otro rol recibe `403`. Documentar ese `403`
   (`ForbiddenRole`) en el bloque `@openapi`; los tests de 403 cruzados de
   `app.test.js` se generan desde el spec.
6. **Test de aislamiento** — al menos un test que verifique que la capa no filtra
   datos de otro `nutricionista_id`.
7. **Docs** — actualizar la documentación de la API (donde esté centralizada en el
   repo) con el endpoint nuevo.

### Cuerpos de entrada en OpenAPI (AUD-005)

- Registrar el schema Zod real en `src/docs/requestSchemas.js` con el nombre que
  usa el `$ref` de la ruta. El cuerpo se genera con `z.toJSONSchema`, para OpenAPI
  3.0, sin repetir campos, tipos, enums ni límites en `components.js`.
- Las descripciones, ejemplos, formatos de presentación y `uniqueItems` de los
  refinamientos personalizados van en `src/docs/requestAnnotations.json`. Una
  anotación no puede reemplazar los tipos ni límites generados.
- El adaptador respeta los opcionales de `optionalField` y los límites de texto
  posteriores a la normalización. Los topes describen el texto normalizado.
- Las reglas cruzadas (`superRefine`), fechas no futuras y límites en bytes no se
  expresan por completo en JSON Schema: mantener su explicación y sus tests Zod.
- Las respuestas, errores y parámetros siguen en `components.js`. No se derivan
  de los validadores de entrada, porque pueden tener otra forma y serialización.

## Reglas que no se negocian

- **Nunca leer `nutricionista_id` del cliente.** Sale siempre del token JWT
  verificado en el middleware de auth, nunca del body/query de la request.
- **Formato de error único** en toda la API: `{ error: { code, message, details? } }`,
  armado solo por `src/middlewares/errorHandler.js`. Un endpoint nunca responde un
  error con `res.status(...).json(...)` a mano: lanza `new AppError(status, message,
  { code, details })`. Los errores de validación llevan el mapa campo → mensaje en
  `details.fields` (ver `docs/decisiones/formato-de-error-uniforme.md`).
- El middleware de aislamiento por `nutricionista_id` se aplica en todas las
  rutas que tocan datos de pacientes — verificar que el endpoint nuevo pasa por
  él, no asumirlo.
- Planes alimentarios son datos estructurados (entidades como `ALIMENTO`,
  `OPCION_ITEM`), no texto libre — no aceptar ni devolver el plan como blob de
  texto donde el modelo de datos define una entidad.
- Las reglas clínicas deterministicas viven en el backend (motor de reglas), no
  se delegan al servicio de IA.

## Test de aislamiento — mínimo esperado

Para cualquier endpoint que lea o escriba datos de paciente: un test que confirme
que un nutricionista no puede ver ni modificar datos de pacientes de otro
nutricionista, incluso conociendo el id del recurso.

## Al terminar

Confirmar que el endpoint completo (las 6 capas) está en un único PR chico y
enfocado en la tarea (`T<N>`), no mezclado con cambios de otro alcance.
