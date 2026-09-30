# Flujo de ramas y convención de commits

**Tarea relacionada:** T10

## Contexto

Tres repositorios con un solo desarrollador, que además trabaja con asistentes de
IA. Hace falta un flujo que mantenga una rama estable para mostrar al cierre de cada
incremento, deje trazada cada tarea del tablero hasta su código y no agregue
ceremonia innecesaria.

## Alternativas consideradas

- **Trunk-based (una sola rama).** Lo más simple. En contra: no hay una rama estable
  que represente "lo que se presenta" al cierre de un incremento.
- **GitFlow completo (release, hotfix, etc.).** Muy estructurado. En contra: pensado
  para equipos y versiones paralelas; para una persona es ceremonia sin beneficio.
- **Dos ramas permanentes más una rama por tarea.** Una rama estable, una de
  integración, y una rama y un PR por cada tarea del tablero.

## Decisión

- **Ramas permanentes:** `master` (estable; recibe la integración al cerrar un
  incremento, con tag `inc-1`, `ia2`, …) y `development` (integración; todos los PR
  apuntan acá). `staging` se crea desde `development` y recibe la iteración completa
  (no cada tarea); es la que se despliega al entorno de validación cuando exista
  (Cloud Run, Incremento 3).
- **Rama por tarea** desde `development`: `T<N>-TituloTarea`, con el número de la
  card de Trello. Un PR por tarea contra `development`.
- **Commits:** `tipo(T<N>): descripción`, con `tipo` ∈ {`feat`, `fix`, `chore`,
  `docs`}.
- Nunca push directo a `master`, `development` o `staging`.
- El historial lo maneja la persona. Los asistentes de IA no hacen commits, push ni
  PR salvo pedido explícito (ver
  [asistentes-de-ia-con-contexto-versionado.md](asistentes-de-ia-con-contexto-versionado.md)).

## Consecuencias

- Cada tarea funcional queda trazada: card `T<N>` → rama `T<N>-…` → PR → commits
  `tipo(T<N>)`. En el Incremento 1 así se integraron T5, T6 y T7 en backend y
  frontend (PR #1 a #3).
- GitHub Free no aplica protección de ramas en repos privados. El "nunca push
  directo" depende de la disciplina (y de la CI cuando exista).
- **Cómo se aplicó en el Incremento 1:**
  - Los PR se integraron con merge commit, no con squash.
  - Los cambios de soporte y las correcciones que no correspondían a una tarea
    funcional (Swagger, pgAdmin, validación de fechas, página 404, Tailwind, tests
    del repositorio) se commitearon directo en `development`.
  - `staging` todavía no existe en ningún repo.
  - No hay plantilla de PR versionada (`.github/pull_request_template.md`).
- El procedimiento de referencia (incluido el cierre de incremento) está en el skill
  `flujo-git-nutria`.
