---
name: pruebas-vitest
description: Cómo organizar y escribir tests del backend de NutrIA con Vitest (unitarios e integración contra Postgres), qué casos son obligatorios y qué otros tipos de prueba define la estrategia del proyecto. Usar al agregar o modificar lógica de negocio en el backend, o al decidir qué prueba cubre algo.
---

# Pruebas — backend (Vitest)

Este skill opera dentro de `nutria-backend/`. El runner es Vitest.

## Estrategia del proyecto

Cada objetivo tiene su tipo de prueba, su comando y su momento. El detalle y su
fundamento están en `docs/decisiones/estrategia-de-pruebas.md`, en el repo raíz.

| Tipo | Comando (desde la raíz) | Cuándo | Estado hoy |
|---|---|---|---|
| Unitarias (backend y frontend) | `npm run test:backend`, `npm run test:frontend` | Cada cambio | **Implementadas** |
| Lint del frontend (oxlint) | `npm run lint:frontend` | Cada cambio | Previsto: el comando no existe todavía en la raíz |
| Integración del backend contra Postgres | `npm run test:backend:integration` | Al cerrar cada tarea del backend | Previsto |
| E2E de camino feliz (Playwright) | `npm run test:e2e` | Al cerrar tareas que toquen esos flujos, y el incremento | Previsto |
| Carga y estrés (k6) | `npm run test:load` | Al cerrar el incremento, y en staging | Previsto |

**No inventar un comando previsto como si existiera.** Hasta que una tarea lo
implemente, no hay que correrlo ni citarlo como verificación. Lo que no cubre una
prueba implementada se informa como verificación manual pendiente.

El backend no lleva lint por ahora (ver la decisión).

**No se versionan scripts de verificación sueltos** (recorridos de la API, seeds
de volumen, mediciones). Si hace falta uno para verificar algo durante una tarea,
va al scratchpad y no al repo. Si lo que verifica tiene que quedar, se convierte
en una prueba del tipo que corresponda.

## Cómo correrlos

Siempre dentro del contenedor, desde la raíz `nutria/` (no desde `nutria-backend/`):

```bash
npm run test:backend             # suite completa, una vez
npm run test:backend:coverage    # + cobertura: tabla en terminal y reporte en nutria-backend/coverage/ (index.html, lcov.info)
```

Eso hace `docker compose run --build --rm --no-deps backend npm run test`:
`--build` rebuildea la imagen (con cache de capas) antes de correr, así que un
test nuevo o un cambio en el código se ve sin ningún paso manual — el rebuild
cacheado tarda segundos porque `npm install` no se repite salvo que
`package.json`/`package-lock.json` hayan cambiado. `--no-deps` porque la suite
unitaria no toca Postgres (repositorios fake). No hay ningún archivo de compose
extra ni mounts: es el compose normal de `nutria-backend/` con dos flags.

Correr `npm run test` directo en `nutria-backend/` sirve para iterar rápido (o
`npm run test:watch`), pero la verificación que cuenta para cerrar una tarea es la
del contenedor.

Los archivos de test son CommonJS como el resto del repo y usan `describe`/`it`/
`expect`/`vi` como globales (`globals: true` en `vitest.config.js`): no hacer
`require('vitest')`, falla.

## Organización

- Un archivo de test por módulo de servicio/repositorio, junto al código que
  prueba (o en la carpeta de tests que ya exista en el repo — no crear una
  estructura paralela nueva).
- **Repositorios fake** en vez de pegarle a la base real en tests unitarios: el
  servicio se testea contra una implementación en memoria del repositorio, no
  contra Prisma + Postgres. Los tests que sí necesitan la base van aparte
  (integración), no se mezclan en la misma suite que corre en cada commit.

## Integración contra Postgres (prevista)

Convenciones definidas para cuando se implemente. Hasta entonces no existe ni la
config ni el comando.

- Archivos `*.int.test.js` junto al módulo, con un `vitest.integration.config.js`
  propio. La config unitaria (`vitest.config.js`) los excluye.
- Base `nutria_test` en el mismo Postgres del compose, nunca la de desarrollo.
  Cada corrida arranca con `prisma migrate reset --force --skip-seed`.
- Datos con fixtures propias armadas con Prisma, no con el seed. Cada archivo vacía
  las tablas antes de cada test, y los archivos corren en serie.
- La app real en un puerto efímero, como `src/app.test.js`, pero con los
  repositorios reales.
- Qué va acá, y no en una unitaria:
  - Aislamiento por nutricionista verificado en la base: el otro nutricionista
    recibe 404 y los datos quedan iguales.
  - `CHECK` y FK de las migraciones, y transacciones.
  - Reglas que dependen de datos guardados.
  - Validación de cada respuesta contra su schema de Swagger.
- Un test aparte verifica que el seed corre sobre base vacía.

## Casos obligatorios (cuando el endpoint/servicio los involucra)

- **Aislamiento por nutricionista:** un nutricionista no puede leer ni modificar
  datos de pacientes de otro nutricionista.
- **Auth:** rutas protegidas rechazan requests sin token válido o con token de
  otro usuario.
- **Guardado parcial:** un plan o formulario que se guarda incompleto (borrador)
  se recupera tal cual se dejó, sin perder datos ni completarlos con valores
  inventados.
- **Menor sin responsable:** alta o edición de un paciente menor de edad sin
  responsable asociado se rechaza (o exige el dato), según la regla vigente —
  no inventar el comportamiento si no está definido, preguntar.
- **Login:** credenciales válidas autentican, inválidas no, y no hay fuga de
  información sobre si el error fue el usuario o la contraseña.

## Al terminar una tarea

El PR de la tarea no se considera completo sin al menos los tests obligatorios
que apliquen a lo que tocó. Si la tarea no toca ninguno de esos casos, el PR debe
igual incluir un test que cubra el camino principal nuevo.

Además, por cada tipo de la tabla de arriba que ya esté implementado:

- **Integración:** si la tarea agrega o cambia un endpoint, una migración o una
  transacción, suma o ajusta su `*.int.test.js`, y la suite corre en verde.
- **E2E:** si la tarea cambia uno de los flujos cubiertos, el E2E sigue en verde.
- **Carga:** corre al cerrar el incremento, no en cada tarea.
