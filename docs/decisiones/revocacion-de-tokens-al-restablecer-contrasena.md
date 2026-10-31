# Revocación de tokens al restablecer la contraseña

**Fecha:** 30/10/2026
**Hallazgo relacionado:** AUD-016

## Contexto

El panel ya restablece contraseñas, pero los JWT emitidos antes seguían válidos
hasta su vencimiento (2 h por defecto). El middleware consulta rol y estado de la
cuenta en cada request; esa lectura permite comprobar también una revocación.

La auditoría propone comparar `iat` con una fecha de cambio. El `iat` emitido
actualmente tiene precisión de segundos: un token anterior y uno posterior al
restablecimiento pueden tener el mismo valor. La corrección debe resolver también
dos restablecimientos dentro del mismo segundo, sin esperas ni desfases de reloj.

## Alternativas consideradas

- **Fecha de cambio e `iat`.** Requiere definir qué sucede con todos los tokens
  del mismo segundo o introducir un dato adicional con mayor precisión.
- **Lista de tokens revocados o tabla de sesiones.** Agrega almacenamiento y
  limpieza que no hacen falta para revocar todas las sesiones de una cuenta.
- **Versión de tokens por cuenta.** Una columna entera y una comparación en la
  lectura que el middleware ya hace; no depende del reloj.

## Decisión

Agregar `nutritionist.token_version`, entero obligatorio con default cero, y
emitirlo en el claim firmado `tokenVersion`. Cada restablecimiento incrementa la
versión en la misma escritura que reemplaza el hash. El middleware acepta solo
la versión vigente de una cuenta activa, con el mismo 401 uniforme ante rechazo.

La regla cubre el panel, la rotación por `create-admin` y la actualización de
contraseñas de las dos cuentas ficticias del seed. Se usa el incremento atómico
de Prisma para no perder revocaciones concurrentes.

## Consecuencias

- La migración agrega el campo con default cero sin borrar cuentas ni datos.
  Los JWT anteriores sin claim se interpretan como versión cero: conservan su
  vigencia hasta que venza el token o se restablezca esa contraseña.
- Dos JWT con igual `iat` pueden distinguirse por la versión, incluso tras dos
  restablecimientos en ese segundo. El cambio afecta la siguiente autorización.
- Un login que leyó la cuenta antes de un restablecimiento puede devolver un
  token de versión anterior; ese token se rechaza y el usuario debe iniciar
  sesión con la contraseña vigente.
- La versión no se agrega a los datos públicos de la cuenta ni se acepta desde
  el body. El frontend ya cierra la sesión ante un 401 y conserva su contrato.
- No se agregan consultas por request, dependencias ni una tabla de sesiones.
