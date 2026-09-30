# Tailwind CSS v4 como framework de estilos del frontend

**Fecha:** 27/09/2026
**Tarea relacionada:** T13

## Contexto

Las pantallas del Incremento 1 (layout, login, pacientes; T5, T6, T7) se hicieron con
un `.css` por componente, a mano y con nombres BEM. Los valores del Figma estaban
copiados en cada archivo: el verde de marca `#006e4e` en más de quince reglas, cuatro
estilos de avatar idénticos, tres estilos de input iguales. No había un lugar único
para los tokens de diseño. Cambiar un color o un radio implicaba buscarlo en todos los
archivos, y cada pantalla nueva sumaba otro `.css` con más copias.

El frontend lo implementa una sola persona a partir del Figma. Los criterios fueron:
cercanía al Figma, esfuerzo, tamaño del bundle y mantenibilidad.

## Alternativas consideradas

- **CSS Modules + variables CSS, sin framework.** No suma dependencias y resuelve el
  scope por componente. En contra: sigue siendo CSS escrito a mano, y los tokens se
  respetan solo si cada uno se acuerda de usar la variable. No trae una escala de
  espaciados ni utilidades de layout.
- **Librería de componentes (MUI, Chakra).** Trae botones, inputs y tablas listos. En
  contra: tienen su propio lenguaje visual, y acercarse al Figma implica pelear con el
  tema de la librería componente por componente. Suman runtime JS (CSS-in-JS en MUI) y
  un bundle bastante más pesado para una app de pocas pantallas.
- **Tailwind CSS v4.** Los valores del Figma pasan casi 1:1 a utilidades
  (`rounded-xl`, `gap-3`, `px-5`), y los que no están en la escala por defecto se
  definen como tokens en un solo bloque `@theme`. No tiene runtime JS: el CSS final
  incluye solo las clases que se usan. Se integra con Vite mediante un plugin oficial
  (`@tailwindcss/vite`), sin config de PostCSS aparte.

## Decisión

Se adopta **Tailwind CSS v4** con `@tailwindcss/vite`. Los tokens del Figma (colores,
tamaños de texto que no están en la escala de Tailwind, sombra de card) viven en el
`@theme` de `src/index.css`. Salieron de los valores que ya usaba el CSS de T5/T6,
que venían del Figma. Radios y espaciados usan la escala por defecto, que ya coincide
con el Figma (`rounded-xl` = 12px, `rounded-2xl` = 16px, múltiplos de 4px).

Las pantallas usan utilidades directamente en el JSX. Lo que se repite se extrae a un
componente React (`Button`, `Avatar`, `BackButton`, `Badge`, `SectionCard`,
`FormField` con su `inputClass`), no a clases CSS con `@apply`.

## Consecuencias

- Hay un solo `.css` en el proyecto (`src/index.css`, con tokens y base). Se
  eliminaron los once `.css` por componente, y al borrar un componente no queda CSS
  muerto.
- El JSX tiene más clases. La legibilidad se sostiene extrayendo componentes o
  constantes de clases (`TH`, `TD`, `FORM_GRID`) cuando una combinación se repite.
- Tamaños de texto que eran casi iguales se unificaron: 13, 13.5, 13.6 y 13.8px pasan
  a `text-body-sm`. Quedan como valores arbitrarios (`text-[28px]`, `tracking-[1.4px]`,
  etc.) solo los que aparecen una única vez en el Figma. Si un valor arbitrario se
  repite, pasa a ser un token.
- El preflight de Tailwind resetea los estilos por defecto del navegador (márgenes de
  títulos, `img` como bloque, botones sin estilo). Cada elemento declara sus estilos
  explícitamente.
- Tailwind v4 usa un binario nativo (`@tailwindcss/oxide`). `npm install` instala el
  que corresponde a cada plataforma (Windows en el host, musl en `node:22-alpine`).
- Las convenciones quedan en el skill `frontend-react-nutria`.
