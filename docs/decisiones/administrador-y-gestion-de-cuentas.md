# Rol administrador y gestión de cuentas desde la aplicación

**Fecha:** 26/10/2026
**Tarea relacionada:** T31

## Contexto

Hasta ahora las cuentas de nutricionista se creaban solo con el seed
([alta-de-nutricionistas-por-seed.md](alta-de-nutricionistas-por-seed.md), DA-07). Para
que las nutricionistas prueben en staging hacen falta cuentas propias, que se puedan dar
de alta, desactivar y reactivar, y cuya contraseña se pueda restablecer sin tocar el
servidor. Además, la curación del catálogo común (T25) necesita el rol administrador que
ya anticipaba [abm-de-catalogos-comunes.md](abm-de-catalogos-comunes.md). Staging es
público y los repos también, así que ninguna credencial puede quedar versionada (RNF-06).

## Alternativas consideradas

- **Tabla de administradores separada, con su propio login.** Aísla del todo los dos
  roles. En contra: duplica el login, el JWT y el middleware para una sola cuenta.
- **Rol en la misma tabla `nutritionist`, con un único login (elegida).** Un solo
  circuito de autenticación. El riesgo es que una ruta nueva quede abierta a los dos
  roles; se cubre con un middleware por rol obligatorio y con una prueba que recorre todo
  el spec.
- **Crear el administrador en el seed.** Simple. En contra: el seed tiene una contraseña
  de desarrollo versionada y además borra y recrea los datos de sus cuentas.
- **Para el token vigente de una cuenta desactivada: aceptar la ventana hasta que venza
  (2 h, DA-04).** No toca la base en cada request. En contra: una cuenta dada de baja
  sigue operando hasta dos horas.
- **Cortar el token en `requireAuth` consultando la cuenta (elegida).** Agrega una
  lectura por clave primaria en cada request autenticada. A cambio, la baja tiene efecto
  inmediato y el rol que autoriza sale de la base.

## Decisión

- `nutritionist` suma `role` (`nutritionist` | `admin`, por defecto `nutritionist`),
  `active` (por defecto `true`) y `created_at`. La baja de una cuenta es lógica, porque
  sus alimentos y recomendaciones privados la referencian.
- **Login único.** El JWT lleva `sub` y `role`; el frontend usa `role` para decidir a qué
  inicio llevar. Una cuenta desactivada recibe el mismo 401 "Credenciales inválidas" que
  una contraseña incorrecta.
- **`requireAuth` lee `{ role, active }` de la cuenta en cada request.** Si la cuenta no
  existe o está desactivada, responde 401 y el frontend lo muestra como sesión expirada.
  El rol que autoriza es el de la base, no el del token. Esto actualiza DA-04: el token
  sigue durando 2 h, pero ya no es la única condición para operar.
- **Denegación por defecto.** Toda ruta protegida usa `requireNutritionist` o
  `requireAdmin`, nunca `requireAuth` suelto. El administrador recibe 403 en pacientes,
  condiciones, consultas, mediciones, recetas, recomendaciones y catálogos (RNF-05), y el
  nutricionista recibe 403 en `/admin/*`. Hasta T25 el administrador tampoco lee el
  catálogo.
- **API de administración** (`/admin/nutritionists`): listado (nombre, email, estado y
  fecha de alta, sin datos de pacientes), alta con contraseña inicial, desactivación y
  reactivación, y restablecimiento de contraseña. Solo opera sobre cuentas de
  nutricionista: el id de un administrador responde 404. El administrador no puede
  desactivarse a sí mismo (403).
- **El administrador se crea con `scripts/create-admin.js`**, que lee `ADMIN_EMAIL`,
  `ADMIN_PASSWORD` y `ADMIN_NAME` (opcional) del entorno. Es idempotente: si la cuenta
  existe, le pone al día la contraseña y la reactiva. Nunca convierte en administrador a
  una nutricionista. Las credenciales no aparecen en el repo, el README ni el CHANGELOG.
- **Contraseñas** de al menos 8 caracteres y hasta 72 bytes (lo que bcrypt considera).
  El email se guarda y se busca en minúsculas y sin espacios.
- **Rate limit en `POST /auth/login`:** 10 intentos fallidos por IP cada 15 minutos
  (configurable). Los exitosos no cuentan. El contador vive en memoria del proceso, lo
  que alcanza con la única instancia de staging. Como el backend corre detrás de
  Traefik, Express confía en un salto de proxy (`trust proxy`) para contar por la IP real
  del cliente.

## Consecuencias

- **Modifica DA-07.** El seed sigue creando sus dos nutricionistas de prueba, pero ya no
  es la única vía de alta. Tampoco toca las cuentas creadas desde el panel: solo actualiza
  las suyas, por email. Un redeploy (`deploy-staging.sh`) no borra volúmenes ni corre el
  seed.
- Una consulta más por request autenticada. Es una búsqueda por clave primaria y no
  cambia el orden del tiempo de respuesta.
- Los tokens emitidos antes del cambio no traen `role`. El backend los acepta, porque el
  rol sale de la base, pero el frontend los descarta y pide iniciar sesión de nuevo.
- Las pruebas de 403 cruzados se generan desde el spec OpenAPI, así que una ruta nueva
  queda cubierta sola. `openapi.test.js` obliga a documentar el 403 en toda operación
  protegida.
- Si en producción hubiera más de una instancia, el contador del rate limit tiene que
  pasar a un store compartido (por ejemplo, Redis).
- **Fuera de alcance:** autorregistro, recuperación de contraseña por email y cambio de
  contraseña por la propia nutricionista. La contraseña inicial y la restablecida se
  comunican por fuera del sistema.
- Requerimiento fuera del catálogo del IA1: se informa en el IA3. Trazabilidad: RNF-05,
  RNF-06; DA-04 (actualizada), DA-07 (modificada); riesgos 3 y 9 del Plan.
