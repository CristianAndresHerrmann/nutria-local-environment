# Muestras de generación del plan

Salidas reales de `POST /plan-proposals` guardadas como evidencia para el informe y como
línea de base: cuando se implemente RAG, se repite la misma solicitud y se compara contra
estas. Cada muestra lleva la fecha en el nombre de archivo.

## 01/11/2026 — sin RAG (T36)

Primera corrida contra los proveedores reales, para el criterio de cierre de T36 y como
insumo de DA-12 ([ficha](../decisiones/adaptadores-de-proveedores-de-ia.md)).

### Condiciones

| | |
|---|---|
| Recorrido | `nutria-ia` con los adaptadores de T36, llamado directo desde Swagger (sin backend ni motor de reglas) |
| Plantilla | `plan_proposal_v1` |
| Razonamiento | `low` en los dos proveedores |
| Solicitud | [`2026-11-01-solicitud.json`](2026-11-01-solicitud.json): el ejemplo del contrato, con datos ficticios. Celiaquía y preferencia por el pescado; 6 grupos, 8 alimentos y 1 receta candidata. Reemplaza al caso de T41, que todavía no existe. |
| RAG | No |

### Corridas

| Proveedor | Modelo | Resultado | Duración | Tokens de entrada | Tokens de salida | Respuesta guardada |
|---|---|---|---|---|---|---|
| OpenAI | `gpt-6-luna` | 200, cumple el contrato | 9 870 ms | 1 769 | 845 | [completa](2026-11-01-openai-gpt-6-luna.json) |
| OpenAI | `gpt-6-luna` | 200, cumple el contrato | 8 196 ms | 1 769 | 817 | Solo métricas (log del servicio) |
| Gemini | `gemini-3.8-flash` | 200, cumple el contrato | 6 727 ms | 1 677 | 1 107 | Solo métricas (log del servicio) |
| Gemini | `gemini-3.8-flash` | 200, cumple el contrato | 8 091 ms | 1 677 | 1 111 | [completa](2026-11-01-gemini-3.8-flash.json) |

Las respuestas guardadas omiten `generation.raw_response`: es la misma propuesta
serializada, en una línea en OpenAI e indentada en Gemini.

### Propuestas

Códigos de la solicitud: G1 Frutas, G2 Verduras, G4 Cereales sin TACC; A1 Manzana, A2
Zanahoria, A3 Zapallito, A4 Pollo, A5 Merluza, A6 Arroz, A7 Yogur natural, A8 Té; R1 Tarta
de verduras sin TACC (receta, 40 min).

**OpenAI `gpt-6-luna`**

| Comida | Opciones |
|---|---|
| Desayuno | (1) Cereales sin TACC (grupo), yogur natural, té · (2) Cereales sin TACC (grupo), manzana, té |
| Almuerzo | (1) Merluza, arroz, zanahoria, zapallito · (2) Pollo, arroz, zanahoria, zapallito · (3) Tarta de verduras sin TACC. Indicación: llevarlo en un recipiente cerrado, en la heladera del trabajo, y recalentarlo en el microondas. |
| Merienda | (1) Yogur natural, manzana · (2) Té, cereales sin TACC (grupo) |
| Cena | (1) Merluza, zanahoria, zapallito, arroz · (2) Pollo, zanahoria, zapallito, arroz · (3) Tarta de verduras sin TACC |

**Gemini `gemini-3.8-flash`**

| Comida | Opciones |
|---|---|
| Desayuno | (1) Yogur natural, manzana · (2) Té, frutas (grupo) |
| Almuerzo | (1) Pollo, arroz, zanahoria · (2) Tarta de verduras sin TACC. Indicación: llevarlo en un recipiente hermético, en la heladera del trabajo, y calentarlo en el microondas. |
| Merienda | (1) Té, manzana · (2) Yogur natural, frutas (grupo) |
| Cena | (1) Merluza, zapallito, zanahoria · (2) Merluza, arroz, verduras (grupo) |

### Observaciones

Lectura manual de las dos respuestas guardadas. No es la salida del motor de reglas ni una
evaluación clínica.

| | OpenAI | Gemini |
|---|---|---|
| Códigos inventados, cantidades, ítems de texto libre | Ninguno | Ninguno |
| Opciones por comida | 2 o 3 | 2 |
| Usa la receta candidata | Almuerzo y cena | Almuerzo |
| Considera el tiempo de cocina para la receta (RF-38) | Sí: avisa que la tarta lleva más tiempo que el disponible | No lo menciona |
| Nombra la condición clínica en los textos | Sí: "celiaquía" en el diagnóstico situacional y "sin TACC" en las recomendaciones generales | No; alude a los "sellos oficiales" de los productos |
| Opciones con la composición esperada (DA-29) | 5 de 8 | 2 de 7 |

- **Mención de la condición clínica.** Es el riesgo residual de contenido clínico en los
  textos libres que dejó abierto DA-28: la plantilla prohíbe las afirmaciones clínicas,
  pero el modelo igual nombra la condición.
- **Composición.** Se cuentan las opciones sin receta, con las categorías de la solicitud y
  los criterios de DA-29: proteína e hidrato en desayuno y merienda; verdura, proteína e
  hidrato en almuerzo y cena. Las opciones de receta no se evalúan, porque la solicitud no
  trae sus ingredientes. Lo que falta en cada proveedor:
  - OpenAI: proteína en la opción 2 del desayuno y en la 2 de la merienda; hidrato en la 1
    de la merienda.
  - Gemini: hidrato en las dos de desayuno, las dos de merienda y la 1 de la cena; proteína
    en la 2 del desayuno y en la 1 de la merienda.

  En el sistema, estas faltas serían advertencias del motor, no bloqueos (RF-44). Los
  criterios de desayuno y merienda son un supuesto pendiente de la entrevista de
  validación.
- **Una sola corrida guardada por proveedor.** Sirve como evidencia de que la integración
  funciona, no como benchmark: la salida cambia entre corridas.

## 01/11/2026 — desde el frontend, con el catálogo completo (sin RAG)

Generación de un plan desde el editor, con una paciente ficticia del seed. Pasa por el
backend: arma el contexto anonimizado con el catálogo permitido (DA-31), llama al servicio
de IA, guarda el borrador y corre el motor de reglas. Es la muestra más cercana al uso real.

### Condiciones

| | |
|---|---|
| Proveedor | OpenAI `gpt-6-luna`, razonamiento `low`, plantilla `plan_proposal_v1` |
| Solicitud | [`2026-11-01-frontend-solicitud.json`](2026-11-01-frontend-solicitud.json): el `context_sent` que registró el backend. 229 alimentos, 44 grupos y 15 recetas candidatas. Condiciones: celiaquía (patología), nueces (alergia), leche (intolerancia), zapallo (rechazo) y frutas (preferencia). Cocina, le gusta y tiene 30 minutos por día; almuerza en la oficina, con heladera y microondas. |
| Respuesta | [`2026-11-01-frontend-openai-gpt-6-luna.json`](2026-11-01-frontend-openai-gpt-6-luna.json), exportada de `ai_generation`: la propuesta original, sin las ediciones del profesional (el plan no se modificó después de generarse) |
| Reglas clínicas cargadas | Ninguna: `clinical_rule` está vacía en la base local hasta la entrevista de validación. El motor aplica solo las condiciones directas (R-01) y la composición (DA-29). |
| RAG | No |

### Tiempo y tokens

| | Ejemplo del contrato (8 alimentos) | Frontend (229 alimentos) |
|---|---|---|
| Tiempo del proveedor | 9 870 ms y 8 196 ms | **12 618 ms** |
| Tokens de entrada | 1 769 | 13 238 |
| Tokens de salida | 845 y 817 | 1 203 |

- **El tiempo medido es el del proveedor:** el servicio de IA lo toma desde que envía el
  prompt hasta que recibe la respuesta, y es la mayor parte de la espera. El tiempo total
  que ve el profesional suma el armado del contexto, la transacción del borrador y el motor
  de reglas en el backend, más la red. Ese total no se registró: el backend no guarda la
  duración de las requests. Para medirlo en las próximas corridas, usar DevTools del
  navegador → Red → la request de generación → Tiempo.
- Con 7,5 veces más tokens de entrada, el tiempo del proveedor creció entre un 28 % y un
  54 % respecto de las corridas con el ejemplo.

### Propuesta

| Comida | Opciones |
|---|---|
| Desayuno | (1) Manzana, tostadas de arroz, huevo · (2) Receta: panqueques de avena y banana ("verificá que la avena sea sin TACC") · (3) Frutilla, semillas de chía, té |
| Almuerzo | (1) Receta: ensalada de arroz y atún · (2) Receta: ensalada de garbanzos, choclo y tomate · (3) Pechuga de pollo, arroz integral, brócoli, tomate · (4) Quinoa, merluza, espinaca, morrón. Indicación: recipiente con tapa, heladera de la oficina y microondas. |
| Merienda | (1) Pera, tostadas de arroz, palta · (2) Receta: panqueques de avena y banana · (3) Naranja, semillas de girasol · (4) Receta: tostadas de arroz con palta y huevo duro |
| Cena | (1) Receta: ensalada de garbanzos, choclo y tomate · (2) Merluza, papa, lechuga, rúcula · (3) Receta: arroz con pollo y verduras · (4) Receta: tortilla de papa con tomate ("verificá los ingredientes y la manipulación para evitar contacto cruzado") |

Los seis textos completos están en la respuesta guardada.

### Hallazgos del motor de reglas

`validation_result: warnings`: 3 advertencias de composición y ningún bloqueo.

| Opción | Hallazgo |
|---|---|
| Desayuno 3 | Falta proteína y hidrato |
| Merienda 1 | Falta proteína |
| Merienda 3 | Falta proteína y hidrato |

### Observaciones

| | |
|---|---|
| Códigos inventados, cantidades, ítems de texto libre | Ninguno (31 ítems: 23 alimentos y 8 recetas) |
| Condiciones directas (nueces, leche, zapallo) | Sin bloqueos del motor: no aparecen ni como alimento ni como ingrediente de las recetas |
| Uso de recetas candidatas | 8 de 15 opciones son recetas, en las cuatro comidas |
| Nombra la condición clínica en los textos | No en el diagnóstico ("los alimentos que mencionaste que evitás"); sí en las recomendaciones contextuales y en dos descripciones de opción ("sin TACC", "contacto cruzado") |
| Opciones con la composición esperada (DA-29) | 12 de 15 según el motor (las recetas se evalúan por sus ingredientes) |

- **Avena común para una paciente celíaca.** La receta "Panqueques de avena y banana" lleva
  *Avena arrollada*, no la variante sin TACC, y se propone en desayuno y merienda. El motor
  no la bloquea porque no hay reglas de celiaquía cargadas. El catálogo enviado sí excluye
  lo que las reglas `prohibited` prohíben (DA-31), pero sin reglas no excluye nada. El
  modelo avisó por su cuenta ("verificá que la avena sea sin TACC"). Es el riesgo 1 del
  Plan: hasta que la entrevista defina las reglas de celiaquía, solo lo detecta la revisión
  del profesional.
- **Contenido clínico en textos libres** (riesgo residual de DA-28): las advertencias sobre
  TACC y contacto cruzado son contenido clínico que el motor no valida.

## Cómo comparar con RAG

Para que la comparación sea válida, repetir con:

- La misma solicitud, el mismo modelo y el mismo nivel de razonamiento. Hay dos líneas de
  base: el ejemplo del contrato (`2026-11-01-solicitud.json`, directo al servicio) y la
  paciente del seed (`2026-11-01-frontend-solicitud.json`, por el backend). Con RAG el
  contexto puede sumar fragmentos recuperados: se compara contra la solicitud original.
- El tiempo total además del del proveedor (DevTools del navegador), porque RAG suma la
  recuperación antes de la llamada.
- La plantilla que use RAG anotada con su versión. Si cambia, la diferencia puede venir de
  la plantilla y no de RAG.
- La respuesta completa guardada en esta carpeta con la fecha, y una sección nueva en este
  archivo con la misma tabla de corridas y de observaciones.

Lo ideal es correr además varias generaciones por proveedor y por el backend, con el
catálogo completo: así los hallazgos de composición y de condición los calcula el motor de
reglas y no una lectura manual. Eso es lo que hace `npm run measure:generation` (T41) con
los casos clínicos del seed: la primera corrida sin RAG, del 01/11/2026, está en
[`mediciones-de-generacion/`](../mediciones-de-generacion/README.md) y es la línea de base
de tiempos para comparar.
