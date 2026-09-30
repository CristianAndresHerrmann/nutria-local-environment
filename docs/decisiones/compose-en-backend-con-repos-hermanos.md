# Docker Compose en el backend, referenciando los repos hermanos

**Tarea relacionada:** T3

## Contexto

NutrIA son tres repositorios independientes (`nutria-backend`, `nutria-frontend`,
`nutria-ia`) más la base de datos. Para desarrollar y para la demo hace falta
levantar todo junto con un solo comando, con los servicios hablándose por una red
común. El `docker-compose.yml` tiene que vivir en algún lado y poder construir las
imágenes de los tres repos.

## Alternativas consideradas

- **Un compose por repo.** Cada repo se levanta solo. En contra: tres archivos que
  coordinar (red compartida, puertos, orden de arranque) y ningún comando único para
  el stack completo.
- **Un cuarto repositorio solo de orquestación.** Separa la infraestructura del
  código. En contra: un repo más que clonar y mantener, cuando el stack cabe en un
  solo archivo.
- **El compose en `nutria-backend`, con rutas relativas a los repos hermanos.** El
  backend ya es el centro del sistema: depende de Postgres y del servicio de IA, y
  el frontend depende de él.

## Decisión

El `docker-compose.yml` vive en `nutria-backend` y construye el frontend y el
servicio de IA desde `../nutria-frontend` y `../nutria-ia`. El proyecto de Compose se
llama explícitamente `nutria` (campo `name:`), así los contenedores e imágenes se
nombran igual sin importar el nombre de la carpeta.

## Consecuencias

- Los tres repos se clonan como carpetas **hermanas** y con esos nombres exactos.
  Si están en otra disposición o con otro nombre, el compose no los encuentra. En
  particular, el repo remoto del frontend se llama `nutria-frontned`, así que hay
  que clonarlo indicando la carpeta `nutria-frontend`.
- El stack incluye `proxy` (Traefik, un virtual host por servicio), `frontend`,
  `backend`, `ia`, `postgres` y `pgadmin`, más un servicio `psql` bajo perfil
  `tools` que no se levanta con `up`.
- Cuando más tarde se creó el repo de tooling `nutria/` (T11), el compose se quedó en
  el backend. Los scripts de la raíz (`npm run up`, `test:*`, `db:*`) lo invocan con
  `-f nutria-backend/docker-compose.yml`.
- Tests, migraciones y seed corren siempre dentro de estos contenedores
  (`docker compose run --build`), no con el runtime del host.
- El compose local es el entorno de respaldo que siempre funciona. El despliegue en
  GCP (Incremento 3) se usa para la presentación final.
