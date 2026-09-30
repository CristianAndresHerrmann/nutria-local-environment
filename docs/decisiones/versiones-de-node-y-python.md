# Versiones fijadas de Node y Python

**Tarea relacionada:** T12 (definida junto con la CI)

## Contexto

Backend y frontend corren sobre Node.js y el servicio de IA sobre Python. Si cada
entorno (máquina local, contenedor, CI) usa la versión que tenga a mano, un mismo
código puede comportarse distinto según dónde se ejecute.

## Alternativas consideradas

- **Etiquetas `latest` en las imágenes.** Sin mantenimiento. En contra: la versión
  cambia sola entre builds y los fallos no se pueden reproducir.
- **Versión exacta (incluido el parche).** Lo más reproducible. En contra: hay que
  actualizarla a mano en cada parche de seguridad.
- **Fijar la línea mayor de Node y la menor de Python.** Estable en lo que importa y
  recibe parches al reconstruir la imagen.

## Decisión

- **Node 22** (`node:22-alpine`) en los Dockerfiles de backend y frontend.
- **Python 3.12** (`python:3.12-slim`) en el Dockerfile del servicio de IA.
- Cuando exista la CI, sus workflows usan las mismas versiones.

## Consecuencias

- La versión que vale es la del contenedor. Tests, migraciones y seed corren siempre
  dentro de Docker, así que la versión de Node o Python instalada en el host no
  afecta la verificación.
- Correr un servicio fuera de Docker con otra versión (por ejemplo, un entorno
  virtual local de `nutria-ia` con otro Python) no está cubierto por esta decisión.
- Subir de versión es un cambio en un solo lugar por repo (la línea `FROM`, y el
  workflow de CI cuando exista).
