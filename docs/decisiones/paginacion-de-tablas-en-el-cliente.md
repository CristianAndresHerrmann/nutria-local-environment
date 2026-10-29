# Paginación de las tablas en el cliente, con un componente común

**Fecha:** 29/10/2026
**Tarea relacionada:** T28

## Contexto

Con el catálogo base de T28 (DA-32), la pantalla de Alimentos muestra 232 filas en una
sola tabla. Hacía falta paginarla.

`GET /foods` devuelve un array, y además del listado lo consumen el buscador de
ingredientes de recetas (`FoodPicker`) y el de condiciones del paciente
(`ConditionForm`). La búsqueda por nombre y el filtro por grupo ya se resuelven en el
backend. El volumen objetivo del catálogo común es de 150 a 250 alimentos (DA-21), más
los privados de cada nutricionista.

Las cuatro tablas del frontend (Alimentos, Pacientes, Recetas y Cuentas del
administrador) repetían el mismo marcado y las mismas constantes de estilo. Las
pantallas de catálogo no tienen diseño en Figma.

## Alternativas consideradas

- **Paginar en la API** (`page` y `pageSize`, respuesta `{ items, total, page,
  pageSize }`). Escala a catálogos grandes, pero cambia el contrato de `GET /foods`, obliga
  a adaptar los tres consumidores, Swagger y las pruebas del backend, y no hace falta con
  el volumen actual.
- **Mostrar todo** sin paginar. Sin trabajo, pero la tabla de Alimentos no se puede
  recorrer.
- **Paginar en el cliente** sobre lo que ya devuelve la API, con un componente común para
  las cuatro tablas.

## Decisión

Las tablas se paginan en el cliente, con 10 o 25 filas por página a elección del usuario
(10 por defecto). Un componente de tabla (`DataTable`), un pie de paginación
(`TablePagination`) y un hook (`usePagination`) se usan en las cuatro pantallas. Cambiar
la búsqueda o un filtro vuelve a la primera página. La API no cambia.

## Consecuencias

- Cada pantalla trae todas sus filas en una request, como antes. Con el volumen de DA-21 es
  una respuesta liviana.
- Si un listado llega a miles de filas, se pagina en la API: el componente no cambia, solo
  de dónde salen las filas y el total.
- El pie no se muestra cuando las filas entran en una página (10 o menos).
- La página no se guarda en la URL: al volver de la ficha de un paciente, el listado
  arranca en la página 1. Queda como mejora futura.
- Una tabla nueva usa `DataTable` y `TablePagination` en vez de repetir el marcado.
