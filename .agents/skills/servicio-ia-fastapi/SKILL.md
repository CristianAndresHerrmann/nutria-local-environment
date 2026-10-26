---
name: servicio-ia-fastapi
description: Cómo armar o modificar un endpoint del servicio de IA (FastAPI) de NutrIA — contrato, prompts, trazabilidad y qué datos no se envían al proveedor. Usar al tocar código en nutria-ia.
---

# Servicio de IA — FastAPI

Este skill opera dentro de `nutria-ia/`. El servicio de IA es un asistente del
nutricionista, no un reemplazo: propone, no decide, y no aplica restricciones
clínicas por su cuenta.

## Contrato de cada endpoint

- **Pydantic estricto** en request y response: nada de aceptar o devolver campos
  sueltos sin schema. Si el modelo generativo devuelve algo que no matchea el
  schema esperado, es un error a manejar explícitamente, no algo que se pasa
  igual al backend.
- **Prompts versionados**: cada prompt vive identificado con una versión (por
  ejemplo, en el nombre de archivo o una constante), para poder rastrear qué
  versión generó una respuesta dada.
- **Parseo estricto de la respuesta del modelo**: si la respuesta no matchea la
  estructura esperada, se reintenta o se falla explícitamente — no se intenta
  "adivinar" la estructura con regex ad hoc.

## Qué no se envía al proveedor externo

- Ningún dato personal identificable del paciente (nombre, contacto, etc.).
- Ninguna regla clínica ni resultado del motor de reglas determinístico — ese
  motor vive en el backend y no se replica ni se consulta al modelo generativo.
- Antes de agregar un campo nuevo al payload que se envía al proveedor,
  confirmar explícitamente que no es dato personal ni dato clínico crítico.

## Trazabilidad

Registrar entrada, prompt/versión usada, respuesta del modelo y si fue validada o
no — sin incluir los datos personales del paciente en ese registro (anonimizar o
referenciar por id interno).

## Prohibido

- No inventar ni afirmar reglas clínicas nuevas: las reglas clínicas las define
  el motor determinístico del backend (`REGLA_CLINICA`, `HALLAZGO`), este
  servicio no las genera ni las modifica.
- No afirmar que una respuesta de IA es "clínicamente segura" — eso lo determina
  la validación posterior (motor de reglas + revisión del nutricionista), nunca
  el propio servicio de IA.
- No usar datos reales de pacientes en pruebas o demos del servicio.

## Infraestructura (referencia)

ONNX Runtime en vez de PyTorch (imagen final < 0.5 GB), embeddings locales con
`sentence-transformers` (384 dimensiones) contra `pgvector`. En staging el
servicio no tiene ruta pública: solo lo alcanza el backend por la red interna de
Docker (`docker-compose.staging.yml` de `nutria-backend`) — nunca expuesto público.
