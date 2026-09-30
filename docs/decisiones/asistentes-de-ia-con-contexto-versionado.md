# Asistentes de IA (Claude Code / Codex) con contexto versionado

**Tarea relacionada:** T11

## Contexto

El desarrollo se hace con asistencia de agentes de código (Claude Code y Codex). Sin
un contexto común, cada sesión arranca sin conocer las decisiones del proyecto
(stack cerrado, aislamiento por nutricionista, formato de errores, prohibición de
inventar reglas clínicas o de enviar datos de pacientes al proveedor externo) y hay
que repetirlas, con el riesgo de que el asistente las contradiga.

## Alternativas consideradas

- **Sin contexto compartido.** Cero mantenimiento. En contra: resultados
  inconsistentes entre sesiones y reglas duras que dependen de acordarse de pedirlas.
- **Un `AGENTS.md` por repo.** Contexto al lado del código. En contra: las reglas
  transversales se triplican y se desincronizan; además el VS Code se abre desde la
  carpeta raíz, no desde cada repo.
- **Un repo chico en la carpeta raíz `nutria/` con una única fuente de contexto.**
  Las reglas viven en un solo lugar y los tres repos hijos quedan ignorados en su
  `.gitignore`. Codex identifica la raíz del proyecto por Git, así que la carpeta
  raíz tiene que ser un repo.

## Decisión

- La raíz `nutria/` es un repo Git de tooling que versiona solo contexto y
  documentación transversal (este `docs/decisiones/` incluido).
- `AGENTS.md` es la única fuente de contexto: reglas vigentes, stack, prohibiciones
  y una sección por repo. `CLAUDE.md` solo lo importa (`@AGENTS.md`), así Claude Code
  y Codex leen lo mismo.
- Los procedimientos paso a paso van como **skills** en `.agents/skills/`:
  `flujo-git-nutria`, `ci-github-actions`, `backend-express-capas`,
  `prisma-migraciones`, `pruebas-vitest`, `servicio-ia-fastapi`,
  `frontend-react-nutria`. Claude Code los encuentra por el symlink local
  `.claude/skills → ../.agents/skills`.
- **Alcance del asistente:** desarrollo, control de versiones, código y pruebas. No
  gestiona Trello ni la planificación. Tampoco hace commits, push ni PR por su cuenta:
  deja los cambios sin commitear y el historial lo maneja la persona.

## Consecuencias

- Las reglas duras (aislamiento, no inventar reglas clínicas, no enviar datos
  identificables al proveedor de IA, tests dentro de Docker) se cargan en cada
  sesión sin tener que pedirlas.
- `AGENTS.md` y los skills son documentación viva: si una decisión cambia y no se
  actualizan, el asistente trabaja con información vieja. Mantenerlos al día es parte
  de cerrar cada tarea.
- Lo que GitHub lee por repo (`.github/`: workflows de CI y plantilla de PR) no puede
  vivir en la raíz y queda dentro de cada repo hijo.
- Para el IA2, este esquema forma parte de la metodología de trabajo: el asistente
  propone e implementa, y la persona decide el alcance, revisa y maneja el
  historial.
