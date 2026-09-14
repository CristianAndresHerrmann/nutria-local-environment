---
name: flujo-git-nutria
description: Ramas, commits, PR, merge y cierre de incremento en los tres repos de NutrIA. Usar al empezar o cerrar una tarea, o al abrir/mergear un PR.
---

# Flujo de ramas — NutrIA

Corré todo comando `git`/`gh` dentro del repo afectado (`nutria-backend/`,
`nutria-ia/` o `nutria-frontend/`), nunca en la raíz `nutria/`.

## Ramas permanentes

- `master`: producción/estable. Solo recibe merge desde `staging` (o desde
  `development` mientras no exista entorno de validación) al cerrar un incremento,
  con tag (`inc-1`, `ia2`, ...).
- `development`: integración. Todas las PRs de tareas apuntan acá. CI en verde
  obligatoria antes de mergear.
- `staging`: se crea desde `development` desde el día uno. Recibe fast-forward (o
  un PR sin cambios de contenido) desde `development` cuando la iteración está
  completa — no en cada tarea individual. Es la rama que se despliega al entorno
  de validación cuando exista (Cloud Run, Incremento 3).

## Al empezar una tarea

1. Parado en `development` actualizado (`git checkout development && git pull`).
2. Crear la rama: `git checkout -b T<N>-TituloTarea` (ejemplo:
   `T6-alta-paciente`). Sin prefijo `feat/`, `fix/`, etc. en el nombre de rama,
   salvo que la persona que inicia la sesión pida explícitamente otra convención
   para esa tarea puntual.
3. Trabajar y commitear siguiendo el formato de abajo.

## Commits

Conventional Commits en español, con el identificador de tarea como scope:

```
tipo(T<N>): descripción breve en minúscula, sin punto final
```

`tipo` ∈ `feat`, `fix`, `chore`, `docs`. Ejemplos:

```
feat(T1): alguna funcionalidad genial
fix(T7): corrige timeout al llamar al servicio de IA
chore(T12): agrega workflow de CI
docs(T9): registra decisión de usar Prisma
```

Un commit, un cambio lógico. No mezclar refactors grandes con features en el
mismo commit si se puede evitar.

## Pull request

- Un PR por tarea, siempre contra `development`.
- Nunca push directo a `master`, `development` o `staging`.
- Merge por squash hacia `development` (el historial de `development` queda un
  commit por tarea).
- Usar la plantilla `.github/pull_request_template.md` del repo: qué tarea es, qué
  cambia, cómo probarlo, y el checklist (tests corridos, sin secretos en el diff,
  migración revisada si aplica, decisión técnica anotada si aplica, trazabilidad
  con el requerimiento/riesgo del Plan de Proyecto).
- En GitHub el término correcto es *pull request*; *merge request* es terminología
  de GitLab — no mezclar.

## Cierre de incremento

1. Confirmar que `development` tiene CI en verde y todas las tareas del incremento
   mergeadas.
2. Fast-forward (o PR) de `development` a `staging`.
3. Merge de `staging` (o `development`) a `master`, con tag del incremento
   (`inc-1`, `ia2`, ...). Ese merge/tag es lo que alimenta el Informe de Avance.
4. Verificar que las decisiones técnicas del incremento quedaron registradas en
   `docs/decisiones/` (ver plantilla `_plantilla.md`).

## Protección de ramas

GitHub Free no aplica rulesets ni branch protection en repos **privados** (sí en
públicos). Mientras eso no esté resuelto (ver card en Backlog de Trello), la única
protección es disciplina + CI: no pushear directo, siempre PR, no mergear con CI
en rojo.
