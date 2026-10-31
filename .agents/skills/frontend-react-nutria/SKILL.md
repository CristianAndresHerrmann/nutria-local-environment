---
name: frontend-react-nutria
description: Cómo construir una pantalla del frontend de NutrIA a partir de Figma y el alcance de la tarea. Usar al agregar o modificar una pantalla o componente del frontend.
---

# Pantallas — frontend (React)

Este skill opera dentro de `nutria-frontend/`. El único usuario del sistema es el
nutricionista — el paciente nunca accede a la interfaz directamente; tenerlo en
cuenta al decidir qué mostrar en pantalla vs. qué va solo en un PDF exportado.

## De Figma a componente

1. Partir del diseño en Figma correspondiente a la pantalla de la tarea, no de una
   idea aproximada de cómo debería verse.
2. Implementar únicamente lo que está en el alcance de la tarea `T<N>`. Si Figma
   muestra elementos de otra tarea o de una funcionalidad futura, esos elementos
   se ocultan, no se dejan a medio implementar ni con datos de mentira.
3. Reusar componentes existentes del repo antes de crear uno nuevo equivalente.

## Estilos (Tailwind CSS v4)

Decisión y motivos en `docs/decisiones/framework-de-estilos-tailwind.md`.

- Utilidades de Tailwind directamente en el JSX. No se crean archivos `.css` por
  componente: el único es `src/index.css` (import de Tailwind, `@theme` y una capa
  base mínima).
- Colores, tamaños de texto y sombras salen de los tokens del `@theme` de
  `src/index.css` (`bg-primary`, `text-ink`, `text-muted`, `border-line`,
  `text-body-sm`, `shadow-card`, …). Si el Figma trae un valor que no está, se agrega
  como token. No se escribe un hex o un `text-[13px]` suelto, salvo un valor que
  aparece una única vez en todo el diseño.
- Radios y espaciados: escala por defecto de Tailwind (`rounded-xl` = 12px,
  `rounded-2xl` = 16px; `h-10.5` = 42px).
- Lo que se repite se extrae a un componente React y no a `@apply`: `Button`
  (`variant="primary" | "secondary"`), `Avatar` (`size="sm" | "lg"`), `BackButton`,
  `Badge` (`tone`), `SectionCard`, `FormField` (y `inputClass` para cualquier input
  de texto, sumando padding si lleva ícono: `` `${inputClass} pl-10` ``).
- Variantes: un mapa `variante → clases` dentro del componente (ver `Button.jsx`,
  `Badge.jsx`). Clases condicionales con template string, sin librerías extra
  (`clsx`, `tailwind-merge`). No pasar por `className` una utilidad que pise otra del
  componente (ej. otra altura): el resultado depende del orden del CSS generado, no
  del orden de las clases. Si hace falta, agregar una variante.
- Estados: `hover:`, `focus:`, `disabled:`, y `aria-invalid:border-danger` para
  inputs con error (el input necesita `aria-invalid`).
- Una combinación de clases larga que se repite dentro de un mismo archivo va en una
  constante (`const TD = '...'`).

## Responsive

- Construir desde 360 px con clases base móviles y los breakpoints por defecto
  `sm`, `md`, `lg`, `xl`. Conservar la composición de escritorio desde `xl`:
  tokens, espaciados y tamaños existentes. Sin CSS nuevo, dependencias ni `clsx`.
- Todo dato, badge, aviso, error y acción debe seguir visible o alcanzable.
  No usar `hidden` sin una alternativa equivalente ni `truncate`/`line-clamp`
  para nombres, emails o datos clínicos sin acceso al texto completo. La identidad
  del sidebar conserva su composición de escritorio mediante un `details` que
  despliega nombre/email/rol completos con teclado, mouse o tacto; en el panel
  móvil se muestran directamente completos. Preferir `min-w-0`, `break-words`,
  `flex-wrap` y grillas que comiencen en una columna. Revisar también strings
  largos sin espacios. Ocultar elementos decorativos sí está permitido.
- El layout usa padding progresivo. Debajo de `lg`, la navegación vive en un
  `dialog` modal con el mismo contenido por rol: usuario completo, rol, email,
  navegación, badges y logout. Debe cerrar con overlay, Escape o navegación,
  contener el foco, devolverlo al disparador y restaurar el scroll del body.
- Los listados usan `DataTable`: tarjetas `md:hidden` y tabla
  `hidden md:table`, generadas desde las mismas `columns`, sin `matchMedia`.
  Cada celda móvil lleva su encabezado; las de `hideHeader` van al pie como
  acciones. Conservar `renderExpanded`, callbacks y etiquetas accesibles.
  Los tests deben consultar cada variante por separado. Los formularios que
  se repiten en ambas variantes no deben introducir ids duplicados; compartir
  sus borradores, errores y estado de envío para conservarlos al cruzar `md`.
- Búsquedas y selects ocupan el ancho disponible en móvil; contadores y acciones
  pasan a otra línea. Apilar encabezados y guardar/cancelar debajo de `sm`.
  Graduar las grillas según el espacio real dentro de sus tarjetas, incluyendo
  formularios anidados; un breakpoint de viewport no garantiza espacio interno.
- `Button` define sus tamaños y wrap por variante; no pisarlos por `className`.
  Las acciones con ícono necesitan una etiqueta visible en dispositivos táctiles,
  además del `aria-label`; `title` solo no alcanza. Conservar textos existentes.
- La página no debe tener scroll horizontal. Usar scroll local accesible y
  evidente para tablas de mediciones y `ScrollTabs`; mantener visible la pestaña
  activa al entrar o cambiar. No ocultar el scrollbar para disimular desbordes.
- Calendarios y desplegables deben permitir leer cada opción completa y alcanzar
  sus controles. `DateInput` usa un portal y ajusta su posición al viewport;
  `FoodPicker` permanece en el flujo debajo de `xl`, con scroll vertical local.
  Contemplar bordes del viewport, cambios de tamaño y teclado virtual.
- El banner de entorno hace wrap y reserva su altura cuando está presente,
  también en login; nunca debe impedir alcanzar las últimas acciones.
- Verificar cada ruta y sus estados con datos ficticios en 360, 390, 768, 1024,
  1280 y 1440 px. Separar pruebas automatizadas de revisión visual: jsdom no
  comprueba geometría, breakpoints ni la contención nativa del foco.
  Tests, lint y build se ejecutan dentro de Docker desde la raíz `nutria/`.

## Cliente HTTP

Un único cliente HTTP centralizado (donde ya esté definido en el repo) para todas
las llamadas al backend. No hacer `fetch`/`axios` sueltos por componente — eso
incluye manejo de auth (token), base URL y manejo de errores de red en un solo
lugar.

## Errores

Errores de validación de formulario se muestran por campo, junto al input que
falló, con el mismo texto/estructura que devuelve el backend cuando sea posible
(no inventar mensajes que no coincidan con la validación real del servidor).

## Prohibido

- No mostrar datos crudos del modelo de IA como si fueran definitivos: toda
  sugerencia de plan que venga de IA debe presentarse como editable, nunca como
  resultado cerrado.
- No implementar funcionalidades fuera del alcance de la tarea "porque ya que
  estamos" — eso se registra como mejora futura, no se construye sin pedirlo.
