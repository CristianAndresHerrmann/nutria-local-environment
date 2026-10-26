---
name: flujo-git-nutria
description: Convenciones de ramas, mensajes de commit, PR y cierre de incremento en los tres repos de NutrIA. Consultar solo cuando se pide explícitamente ejecutar o revisar una operación de git/gh, o cuando hay una duda sobre la convención. El asistente no commitea ni abre PRs por su cuenta.
---

# Flujo de ramas — NutrIA

Corré todo comando `git`/`gh` dentro del repo afectado (`nutria-backend/`,
`nutria-ia/` o `nutria-frontend/`), nunca en la raíz `nutria/`.

## Quién maneja el repositorio

El control del historial es de quien inicia la sesión, no del asistente. El
asistente **no** hace `git add`, `git commit`, `git push`, `git merge`, `git
checkout -b` ni abre PRs por iniciativa propia, y tampoco propone cortar el
trabajo en varios commits ni sugiere "conviene commitear acá". Deja los cambios
en el working tree y listo; el commit lo hace la persona cuando quiere y con el
alcance que quiere.

Excepción única: que se le pida explícitamente en esa sesión ("commiteá esto",
"abrí el PR"). Ahí sí ejecuta lo pedido, con el alcance pedido — sin dividir en
commits extra ni agregar pasos que no se pidieron.

El resto de este documento es **referencia de convenciones** (cómo se nombran
ramas, commits y PRs en este proyecto), para consultar o para cuando se pida
ejecutar algo. No es una secuencia que el asistente deba empujar.

## Ramas permanentes

- `master`: producción/estable. Solo recibe merge desde `staging` (o desde
  `development` mientras no exista entorno de validación) al cerrar un incremento,
  con tag (`inc-1`, `ia2`, ...).
- `development`: integración. Todas las PRs de tareas apuntan acá. CI en verde
  obligatoria antes de mergear.
- `staging`: se crea desde `development` desde el día uno. Recibe fast-forward (o
  un PR sin cambios de contenido) desde `development` cuando la iteración está
  completa — no en cada tarea individual. Cada push a `staging` se despliega solo
  en la VM de staging (`cd-staging.yml` de cada repo), así que mergear ahí es
  desplegar.

## Nombre de rama de tarea

Se crean desde `development` actualizado, con el formato `T<N>-TituloTarea`
(ejemplo: `T6-alta-paciente`). Sin prefijo `feat/`, `fix/`, etc. en el nombre de
rama.

## Formato de mensaje de commit

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

Qué entra en cada commit lo decide la persona. No hay regla de granularidad acá
y el asistente no opina sobre eso.

## Pull request

- Un PR por tarea, siempre contra `development`.
- Nunca push directo a `master`, `development` o `staging`.

## Cierre de incremento

Secuencia de referencia; la ejecuta quien lleva el proyecto.

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
