# Registro de decisiones del Incremento 3

Insumo del informe de avance del Incremento 3 (T42). Continúa la numeración DA de
[`registro-de-decisiones-incremento-2.md`](registro-de-decisiones-incremento-2.md)
(DA-15 a DA-24). Cada decisión tiene su fundamento completo en
[`decisiones/`](decisiones/_indice.md).

## Decisiones adoptadas en el Incremento 3

| ID | Decisión | Alternativas consideradas | Motivo y consecuencias |
|---|---|---|---|
| DA-25 | Rol administrador en la misma tabla de cuentas, con un único login. El administrador da de alta, desactiva, reactiva y restablece la contraseña de las cuentas de nutricionista desde la aplicación, sin acceso a datos de pacientes ni al catálogo hasta T25. Lo crea un script idempotente que lee sus credenciales de variables de entorno. Una cuenta desactivada no inicia sesión, y su token deja de servir en la siguiente request, porque cada request autenticada consulta el estado de la cuenta. El login tiene un límite de intentos fallidos por IP. | Tabla de administradores con login propio; administrador creado por el seed; aceptar que un token siga vigente hasta 2 h después de la baja. | Las nutricionistas necesitan cuentas propias para probar en staging, que es público (RNF-06: sin credenciales versionadas). Toda ruta protegida exige un rol, y una prueba generada desde el spec verifica los 403 cruzados (RNF-05; riesgos 3 y 9). Agrega una lectura por clave primaria por request. Requerimiento fuera del catálogo del IA1. Modifica DA-07. Actualiza DA-04. |

## Actualización de decisiones anteriores

| ID | Estado |
|---|---|
| DA-04 | Actualizada por DA-25: el JWT sigue durando 2 h, pero una cuenta desactivada deja de operar en la siguiente request. |
| DA-07 | Modificada por DA-25: el seed ya no es la única vía de alta de nutricionistas. |
