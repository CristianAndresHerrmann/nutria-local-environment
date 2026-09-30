# Autenticación con JWT + bcrypt y aislamiento por nutricionista

**Tarea relacionada:** T5

## Contexto

Cada nutricionista accede con credenciales propias (RF-01) y solo puede ver y
modificar a sus pacientes (RNF-05). Son datos de salud: una fuga entre
nutricionistas es el riesgo más grave del incremento. El frontend (`local.nutria.com`)
y la API (`api.local.nutria.com`) están en orígenes distintos. La base de datos se
usa como Postgres administrado sin servicios propietarios, para no atar el sistema a
un proveedor.

## Alternativas consideradas

- **Sesiones en el servidor con cookie.** Permiten revocar una sesión al instante.
  En contra: requieren un almacén de sesiones, y la cookie entre orígenes distintos
  obliga a configurar CORS con credenciales y `SameSite`.
- **Auth administrado del proveedor de base (Supabase Auth).** Menos código propio.
  En contra: ata la autenticación al proveedor, algo que se descartó para mantener la
  portabilidad.
- **JWT sin estado en el header `Authorization`, contraseñas con bcrypt.** No
  necesita almacén de sesiones, funciona igual entre orígenes, y el backend puede
  verificar el token sin consultar la base.

## Decisión

- `POST /auth/login` valida email y contraseña contra el hash bcrypt (`bcryptjs`,
  JavaScript puro, sin compilación nativa en la imagen Alpine) y emite un JWT
  firmado con `JWT_SECRET`, con `sub` = id del nutricionista y vencimiento
  configurable (`JWT_EXPIRES_IN`, 2 h por defecto).
- Las credenciales inválidas devuelven siempre el mismo `401`. Si el email no
  existe, igual se compara contra un hash ficticio, para que ambos casos tarden lo
  mismo y no se pueda averiguar qué emails están registrados.
- El middleware `requireAuth` verifica el token e inyecta `req.nutricionistaId` a
  partir del payload verificado. **El id del nutricionista nunca se lee del
  cliente:** los schemas de entrada son estrictos y rechazan un `nutritionistId` en
  el body.
- Todos los repositorios de pacientes filtran por `nutritionistId` en el mismo
  `where` de la consulta, incluida la escritura (`updateMany` con `id` y
  `nutritionistId`). Un paciente de otro nutricionista, o un id mal formado,
  responde `404`, igual que uno inexistente.

## Consecuencias

- El aislamiento está en dos capas: el middleware fija la identidad y el
  repositorio la aplica en cada consulta. Tiene tests en las dos: el servicio con un
  repositorio en memoria (id ajeno da `404` al leer y al editar) y el repositorio con
  un cliente Prisma falso que verifica el `where`. Otros tests recorren cada ruta de
  pacientes y verifican que exija token y que pase al servicio el id del token.
- Prisma ignora los filtros con valor `undefined`: un `nutritionistId` vacío
  convertiría el filtro en una búsqueda sobre todos los pacientes. Para que un error
  de cableado no se convierta en una fuga, el middleware rechaza tokens sin `sub` y el
  repositorio corta con un error cualquier consulta sin `nutritionistId`.
- Un token no se puede revocar antes de que venza. Cerrar sesión es descartar el
  token en el cliente. Lo mitiga el vencimiento corto.
- El frontend guarda el token en `localStorage`. Si queda vencido, o si la API
  responde `401`, cierra la sesión y vuelve al login con aviso.
- Cualquier recurso nuevo con datos de paciente tiene que seguir el mismo patrón
  (filtro en el repositorio, `404` para ids ajenos, test de aislamiento). El skill
  `backend-express-capas` lo exige.
