# Alta de nutricionistas solo por seed

**Tarea relacionada:** T5
**Estado:** modificada por [administrador-y-gestion-de-cuentas.md](administrador-y-gestion-de-cuentas.md) (T31). El seed sigue creando sus dos nutricionistas de prueba, pero las cuentas también se administran desde la aplicación.

## Contexto

El sistema necesita al menos un nutricionista para poder iniciar sesión, pero el
catálogo de requerimientos no incluye el registro de usuarios ni su administración.
Resolver el alta dentro de la aplicación implica decidir quién puede crear cuentas y
cómo se controla, lo que agrega roles y pantallas fuera del alcance.

## Alternativas consideradas

- **Registro público.** Rápido de implementar. En contra: cualquiera podría crearse
  una cuenta en un sistema que maneja datos de salud.
- **Panel de administración con rol de administrador.** Resuelve el alta de forma
  controlada. En contra: suma un rol, pantallas y permisos que ningún requerimiento
  pide.
- **Alta por script (seed).** Los nutricionistas se crean fuera de la aplicación,
  con acceso al servidor.

## Decisión

En este incremento los nutricionistas se dan de alta **solo con el seed**
(`scripts/seed.js`, ejecutado con `npm run db:seed`). El seed crea o actualiza **dos**
nutricionistas ficticios, cada uno con sus propios pacientes de prueba. Entre los del
primero hay uno menor de edad con adulto responsable y uno con la ficha incompleta.
Con dos cuentas se puede verificar a mano el aislamiento: cada nutricionista ve y
edita solo a sus pacientes.

## Consecuencias

- No hay registro, administración de cuentas ni recuperación de contraseña en la
  interfaz. Por eso el login oculta "¿Olvidaste tu contraseña?".
- El seed borra y vuelve a crear los pacientes de *sus* nutricionistas en cada
  ejecución. Sirve para dejar la demo en un estado conocido, pero no se debe correr
  sobre datos que se quieran conservar.
- Las credenciales del seed son de prueba, están en `scripts/seed.js` y el script
  las imprime al terminar. No se repiten en README ni CHANGELOG.
- Sumar otro nutricionista implica agregarlo al seed o a un script equivalente. El
  panel de administración queda fuera del catálogo actual.
