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
