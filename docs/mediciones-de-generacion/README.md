# Mediciones de la generación del plan

Varias generaciones por caso clínico ficticio y por proveedor, de punta a punta por el
backend. Son la medición preliminar del Incremento 3 de las dos metas de la Tabla 7 del
Informe de Avances 2, un insumo del benchmark DA-12 y la línea de base de varias corridas
para comparar más adelante: con RAG, con otra plantilla o en staging. Las mediciones
cubren los riesgos 4 (validación con casos definidos), 10 (tiempos objetivo) y 17
(tiempos registrados en pruebas controladas).

Las [muestras de generación](../muestras-de-generacion/README.md) guardan una sola salida
completa y leída a mano. Acá hay muchas corridas resumidas: los hallazgos los calcula el
motor de reglas, no una lectura manual. Cada corrida queda en un JSON con la fecha, el
proveedor y el modelo en el nombre. Por qué es un script y no k6:
[ficha de la decisión](../decisiones/medicion-de-la-generacion.md).

## Cómo correrlo

Con el stack levantado (`npm run up`) y los casos cargados (`npm run db:seed` o
`npm run db:reset`), desde la raíz `nutria/`:

```bash
npm run measure:generation -- --n 3                  # N generaciones por caso
npm run measure:generation -- --n 3 --cases C1,C3    # solo algunos casos
npm run measure:generation -- --n 3 --note "Razonamiento low"
npm run measure:generation -- --n 3 --label razonamiento-none   # sufijo en el nombre del JSON
```

- Mide con el proveedor que tenga configurado `nutria-ia`. Para cambiarlo, editar
  `AI_PROVIDER` y `AI_MODEL` en `nutria-ia/.env` y recrear el servicio:
  `docker compose -f nutria-backend/docker-compose.yml up -d ia`. El nivel de
  razonamiento no queda en la traza: anotarlo con `--note`.
- Corre solo a mano: no forma parte de los tests, de la CI ni del despliegue. Con un
  proveedor real, cada generación tiene costo.
- Cada generación crea un borrador en el paciente del caso, que se puede abrir en el
  editor. `npm run db:reset` los borra.
- Imprime las tablas de abajo, listas para copiar, y escribe el JSON en esta carpeta.

## Qué mide

### El tiempo total: de la solicitud a la propuesta validada

Es el tiempo que formaliza la Tabla 7 del IA2: "desde la solicitud hasta la propuesta
validada disponible en el editor". El script lo mide con `performance.now()` así:

- **Empieza** justo antes de enviar `POST /patients/:id/plans/generate`. Equivale al clic
  en "Generar" del editor.
- **Termina** cuando terminó de recibir y leer la respuesta `201`. En ese momento el
  borrador ya está guardado con sus ítems, sus hallazgos y su traza, y es lo que el editor
  mostraría.

Entre esos dos puntos pasa todo esto, en orden:

| # | Paso | Dónde | Parte |
|---|---|---|---|
| 1 | Autenticación (JWT y estado de la cuenta) y validación de la request | Backend | Overhead |
| 2 | Lectura del paciente, sus condiciones, el catálogo y las reglas `prohibited` | Backend → Postgres | Overhead |
| 3 | Catálogo permitido y contexto anonimizado (DA-31) | Backend | Overhead |
| 4 | Llamada HTTP a `nutria-ia` y armado del prompt con la plantilla versionada | Backend → `nutria-ia` | Overhead |
| 5 | **Llamada al proveedor**: la request por internet a Google u OpenAI, la generación del modelo (con su razonamiento) y la respuesta | `nutria-ia` → proveedor | **Proveedor** |
| 6 | Validación estricta de la salida contra el contrato (DA-28) | `nutria-ia` | Overhead |
| 7 | Validación con Zod, resolución de los códigos a ítems y motor de reglas | Backend | Overhead |
| 8 | Transacción: borrador, ítems, hallazgos y traza | Backend → Postgres | Overhead |
| 9 | Serialización y respuesta | Backend | Overhead |

- **Proveedor** es `ai_generation.duration_ms`, que `nutria-ia` mide solo alrededor del
  paso 5.
- **Overhead** es el total menos el proveedor, o sea los pasos 1–4 y 6–9 más la red interna
  de Docker. Es lo que el sistema controla: el proveedor no.

**Qué no incluye**, y por qué no cambia la conclusión:

- **El navegador y Traefik.** El script corre dentro de la red de Docker y le habla al
  backend directo. Desde el navegador se suman la red hasta el servidor y el render del
  editor: decenas de milisegundos en local, algo más en staging, frente a 12–14 s del
  proveedor.
- **Login y navegación previa.** No son parte de la generación.
- **Reintentos.** No hay: ningún SDK reintenta (DA-28) y el script mide cada intento por
  separado.
- **Concurrencia.** Las generaciones van de a una. Con varios profesionales generando a la
  vez, el backend casi no cambia (unos 110 ms por generación), y el límite lo pone el
  proveedor.

Los tiempos se calculan solo sobre las generaciones válidas (`201`): una falla no llega a
ser una propuesta validada. En estas corridas no hubo fallas.

### El P95

El P95 es el tiempo dentro del cual terminó el 95 % de las generaciones. Se usa en vez del
promedio porque la meta es sobre la espera que tiene el profesional casi siempre, incluidas
las generaciones lentas, que el promedio esconde. Es la formalización de la Tabla 7.

**Cálculo (rango más cercano):**

1. Se ordenan los N tiempos de menor a mayor.
2. El P95 es el valor en la posición ⌈0,95 × N⌉.

Siempre es un tiempo que realmente ocurrió, sin interpolar. Con este método:

| Muestras (N) | Posición del P95 | Qué significa | Para cumplir ≤ 15 s |
|---|---|---|---|
| 21 (7 casos × 3) | 20.ª | El segundo más lento | A lo sumo 1 de 21 por encima de 15 s |
| 35 (7 casos × 5) | 34.ª | El segundo más lento | A lo sumo 1 de 35 |
| 70 (7 casos × 10) | 67.ª | El cuarto más lento | A lo sumo 3 de 70 |
| 140 (7 casos × 20) | 133.ª | El octavo más lento | A lo sumo 7 de 140 |

El P50 (mediana) se calcula igual, en la posición ⌈0,5 × N⌉: con 21 muestras es la 11.ª.

**Lectura:**

- **Con 21 muestras, el P95 es casi el máximo.** Una sola generación lenta lo mueve
  mucho: es indicativo, no definitivo. La medición definitiva es del Incremento 4, con el
  proveedor elegido y más muestras (N=10 o más por caso).
- **Cada columna tiene su propio P95**, calculado sobre sus valores. El P95 del total no es
  la suma del P95 del proveedor y el del overhead, porque la generación más lenta en una
  columna no es necesariamente la más lenta en la otra.
- **Por caso, con 3 muestras,** P95 y máximo coinciden, porque ⌈0,95 × 3⌉ = 3. Las tablas
  por caso sirven para ver la variación, no para evaluar la meta por caso.

### Otras columnas

| Columna | Qué es |
|---|---|
| Válidas | Generaciones que terminaron en `201`: la salida cumplió el contrato y el plan se guardó. Las fallas se cuentan por su `reason` (`invalid_output`, `provider_error`, `provider_timeout`…). |
| Sin bloqueantes | Propuestas válidas con `validation_result` distinto de `blocking`. La meta del 95 % se calcula sobre los intentos: una falla tampoco es una propuesta utilizable. |
| Tokens y costo | De la traza, incluidas las fallas que llegaron al proveedor. Los de salida incluyen los de razonamiento, que desde la prueba de la plantilla v2 también se registran aparte ("De razonamiento"; vacío si el proveedor no los informa). El costo usa la tabla `PRICES` del script (USD por millón de tokens), que anota la fuente y la fecha de cada precio. |
| Opciones y caracteres | El tamaño de cada propuesta válida: la cantidad de opciones y el largo de los seis textos que redacta la IA. Están solo en el JSON, no en la tabla impresa. |

## Casos

Los define `nutria-backend/scripts/seed-data/clinical-cases.js` y el seed los carga en
`nutricionista@nutria.com`. Todo es ficticio. El resultado esperado de cada caso contra
la matriz de reglas de T35 está en la ficha R-05 de
[`reglas-clinicas.md`](../reglas-clinicas.md#casos-de-t41) y lo prueba
`clinical-cases.rules.test.js`. Las corridas de abajo son anteriores a T35: no tienen
reglas por patología cargadas.

| Código | Caso | Paciente | Condición | Contexto |
|---|---|---|---|---|
| C1 | Celiaquía | Sofía Medina | Patología Celiaquía | Cocina poco (20 min); almuerza en la oficina con microondas y sin heladera |
| C2 | Diabetes | Ricardo Benítez | Patología Diabetes | Cocina y le gusta; come en casa |
| C3 | Hipertensión | Elena Quiroga | Patología Hipertensión | **No cocina**: compra comida hecha (solo recetas sin cocción) |
| C4 | Alergia alimentaria | Pablo Herrera | Alergia al grupo Huevos | Cocina; almuerza en el trabajo |
| C5 | Intolerancia | Natalia Vega | Intolerancia a Leche | Cocina y le gusta; trabaja desde la casa |
| C6 | Sin condiciones | Federico Luna | — | Cocina poco; almuerza afuera dos veces por semana |
| C7 | Rechazo de un grupo | Joaquín Ruiz (menor) | Rechazo del grupo Legumbres, con sus subgrupos | Almuerza en el comedor escolar |

## 01/11/2026 — sin RAG, por el backend

### Resumen para el informe

Medición preliminar del Incremento 3 sobre los 7 casos ficticios del Plan 5.6, por el
recorrido completo del sistema, con los dos candidatos de DA-12 (razonamiento `low`,
plantilla `plan_proposal_v1`, sin RAG, sin reglas clínicas cargadas). Se hicieron 21
generaciones por proveedor, 3 por caso.

- **Tiempo hasta la propuesta validada:**
  - Con razonamiento `low`, ninguno de los dos cumple la meta de 15 s en P95: Gemini 3.8
    Flash tardó 17,1 s y GPT-6 Luna 16,4 s. Con `none`, GPT-6 Luna baja a 10,5 s (ver la
    sección siguiente).
  - La mediana está por debajo (13,8 s y 12,1 s): el 81 % y el 86 % de las generaciones
    terminaron en 15 s o menos.
  - El 99 % del tiempo es del proveedor. El sistema agrega unos 110 ms.
- **Validez:** 42 de 42 salidas cumplieron el contrato.
- **Propuestas sin hallazgos bloqueantes:** 95,2 % en los dos (20 de 21). Es una cota
  superior, porque todavía no hay reglas de patologías que puedan bloquear.
- **Costo por generación:** US$ 0,018 con Gemini y US$ 0,002 con OpenAI.

### Condiciones

| | |
|---|---|
| Recorrido | Script → backend → `nutria-ia` → proveedor, en el stack local (Docker Desktop, Windows), recién reseteado con `npm run db:reset` |
| Plantilla | `plan_proposal_v1` |
| Razonamiento | `low` en los dos proveedores reales |
| Comidas | Las cuatro (por defecto) |
| Catálogo enviado | El común completo menos lo excluido por las condiciones de cada caso (DA-31): unos 13 000 tokens de entrada en OpenAI y 16 000 en Gemini, con el mismo prompt |
| Reglas clínicas cargadas | Ninguna (`clinical_rule` vacía hasta T35). Las patologías no generan hallazgos: el motor aplica solo las condiciones directas, los ítems no reconocidos y la composición (DA-29, DA-30). |
| RAG | No |
| Corridas | Simulado: N=5 (35 generaciones), a las 14:21, en menos de 5 s. Gemini: N=3 (21), de 14:22 a 14:27. OpenAI: N=3 (21), de 14:27 a 14:32. Las tres en serie y en round-robin por caso. |

### Resumen por proveedor

| | Simulado | Gemini `gemini-3.8-flash` | OpenAI `gpt-6-luna` |
|---|---|---|---|
| Archivo | [JSON](2026-11-01-fake-fake-deterministic.json) | [JSON](2026-11-01-gemini-gemini-3.8-flash.json) | [JSON](2026-11-01-openai-gpt-6-luna.json) |
| Válidas | 35/35 | 21/21 | 21/21 |
| **Total P50** | 105 ms | 13 839 ms | 12 098 ms |
| **Total P95** (meta ≤ 15 s) | 174 ms | **17 132 ms ✗** | **16 430 ms ✗** |
| Total máximo | 209 ms | 20 607 ms | 17 803 ms |
| Total medio | 107 ms | 13 984 ms | 12 394 ms |
| Proveedor medio | ≈ 1 ms | 13 870 ms | 12 286 ms |
| Overhead medio / P95 | 106 / 172 ms | 114 / 176 ms | 108 / 153 ms |
| **Sin bloqueantes** (meta ≥ 95 %) | No aplica (ver abajo) | **95,2 % ✓** (20/21) | **95,2 % ✓** (20/21) |
| Advertencias de composición | — | 20 en 21 propuestas | 56 en 21 propuestas |
| Tokens de entrada / salida (media) | — | 15 900 / 1 561 | 13 163 / 1 158 |
| Costo por generación | 0 | US$ 0,0178 | US$ 0,0019 |
| Costo de la corrida | 0 | US$ 0,37 | US$ 0,04 |

Precios: Gemini US$ 0,75 de entrada y 3,75 de salida por millón (hasta el 31/12/2026;
desde el 01/01/2027, 1,50 y 7,50); OpenAI US$ 0,10 y 0,50. Consultados el 01/11/2026 en
las páginas oficiales enlazadas en el script.

### Tiempos ordenados

Los 21 tiempos totales de cada proveedor, en milisegundos y de menor a mayor. La mediana
(P50) es el 11.º valor y el P95 es el 20.º (ver [El P95](#el-p95)). En negrita, los que
superan los 15 s.

| Posición | Gemini `gemini-3.8-flash` | OpenAI `gpt-6-luna` |
|---|---|---|
| 1.º–5.º | 10 981 · 11 161 · 12 018 · 12 307 · 12 592 | 8 487 · 9 541 · 10 161 · 10 311 · 10 476 |
| 6.º–10.º | 12 781 · 12 808 · 13 555 · 13 636 · 13 693 | 10 772 · 11 073 · 11 428 · 11 579 · 12 046 |
| **11.º (P50)** | **13 839** | **12 098** |
| 12.º–17.º | 13 944 · 14 072 · 14 145 · 14 663 · 14 733 · 14 863 | 12 188 · 12 504 · 12 548 · 13 333 · 13 935 · 14 037 |
| 18.º–19.º | **15 060** · **15 075** | 14 420 · **15 102** |
| **20.º (P95)** | **17 132** | **16 430** |
| 21.º (máximo) | **20 607** | **17 803** |
| Por encima de 15 s | 4 de 21 (19 %) | 3 de 21 (14 %) |

Para cumplir la meta con 21 muestras, a lo sumo una generación puede superar los 15 s.
Gemini tiene cuatro, dos de ellas por menos de 0,1 s. OpenAI tiene tres. El P95 lo fija
la segunda generación más lenta, así que con esta muestra basta una generación lenta de
más para que no se cumpla.

### Simulado (`fake-deterministic`)

| Caso | Intentos | Válidas | Sin bloqueantes | Total P50 (ms) | Total P95 (ms) | Total máx. (ms) | Proveedor P95 (ms) | Overhead P95 (ms) |
|---|---|---|---|---|---|---|---|---|
| C1 | 5 | 5 | 0 | 109 | 209 | 209 | 1 | 208 |
| C2 | 5 | 5 | 0 | 98 | 129 | 129 | 1 | 128 |
| C3 | 5 | 5 | 0 | 114 | 174 | 174 | 2 | 172 |
| C4 | 5 | 5 | 0 | 106 | 130 | 130 | 2 | 128 |
| C5 | 5 | 5 | 0 | 95 | 111 | 111 | 1 | 110 |
| C6 | 5 | 5 | 0 | 106 | 119 | 119 | 1 | 118 |
| C7 | 5 | 5 | 0 | 95 | 121 | 121 | 1 | 121 |
| **Global** | 35 | 35 | 0 | 105 | 174 | 209 | 2 | 172 |

El simulado no participa de la elección del proveedor, que se decide con los proveedores
reales. Sirve para dos cosas:

- **Comprobar que el recorrido completo funciona** sin costo: 35 de 35 generaciones
  válidas, con borrador, hallazgos y traza.
- **Medir lo que agrega el sistema.** Sin proveedor, la generación completa tarda unos
  100 ms (P95 174 ms). Ese es el overhead que también aparece en las corridas reales.

Su columna "Sin bloqueantes" (0 %) no aplica. El simulado agrega a propósito un ítem de
texto libre ("Agua") en cada propuesta para ejercitar el ítem no reconocido (DA-26), así
que todas tienen un bloqueante por diseño. Tampoco informa tokens.

### Gemini `gemini-3.8-flash`

| Caso | Intentos | Válidas | Sin bloqueantes | Total P50 (ms) | Total P95 (ms) | Total máx. (ms) | Proveedor P95 (ms) | Overhead P95 (ms) | Tokens entrada (media) | Tokens salida (media) | Costo |
|---|---|---|---|---|---|---|---|---|---|---|---|
| C1 | 3 | 3 | 3 | 14 733 | 20 607 | 20 607 | 20 429 | 178 | 16 259 | 1 589 | US$ 0,0545 |
| C2 | 3 | 3 | 3 | 14 072 | 14 663 | 14 663 | 14 549 | 176 | 16 273 | 1 635 | US$ 0,0550 |
| C3 | 3 | 3 | 2 | 13 693 | 15 075 | 15 075 | 14 954 | 121 | 15 358 | 1 558 | US$ 0,0521 |
| C4 | 3 | 3 | 3 | 12 808 | 14 145 | 14 145 | 14 010 | 143 | 15 754 | 1 460 | US$ 0,0519 |
| C5 | 3 | 3 | 3 | 14 863 | 17 132 | 17 132 | 17 027 | 161 | 16 045 | 1 600 | US$ 0,0541 |
| C6 | 3 | 3 | 3 | 13 944 | 15 060 | 15 060 | 14 932 | 128 | 16 243 | 1 655 | US$ 0,0552 |
| C7 | 3 | 3 | 3 | 11 161 | 12 781 | 12 781 | 12 674 | 107 | 15 370 | 1 433 | US$ 0,0507 |
| **Global** | 21 | 21 | 20 | 13 839 | 17 132 | 20 607 | 17 027 | 176 | 15 900 | 1 561 | US$ 0,3734 |

### OpenAI `gpt-6-luna`

| Caso | Intentos | Válidas | Sin bloqueantes | Total P50 (ms) | Total P95 (ms) | Total máx. (ms) | Proveedor P95 (ms) | Overhead P95 (ms) | Tokens entrada (media) | Tokens salida (media) | Costo |
|---|---|---|---|---|---|---|---|---|---|---|---|
| C1 | 3 | 3 | 3 | 16 430 | 17 803 | 17 803 | 17 626 | 177 | 13 457 | 1 490 | US$ 0,0063 |
| C2 | 3 | 3 | 3 | 9 541 | 10 476 | 10 476 | 10 409 | 153 | 13 476 | 871 | US$ 0,0053 |
| C3 | 3 | 3 | 2 | 11 579 | 12 548 | 12 548 | 12 458 | 127 | 12 690 | 1 149 | US$ 0,0055 |
| C4 | 3 | 3 | 3 | 12 188 | 13 333 | 13 333 | 13 190 | 143 | 13 042 | 1 194 | US$ 0,0057 |
| C5 | 3 | 3 | 3 | 12 504 | 14 037 | 14 037 | 13 932 | 150 | 13 282 | 1 084 | US$ 0,0056 |
| C6 | 3 | 3 | 3 | 12 046 | 12 098 | 12 098 | 12 008 | 112 | 13 444 | 1 103 | US$ 0,0057 |
| C7 | 3 | 3 | 3 | 13 935 | 14 420 | 14 420 | 14 307 | 113 | 12 749 | 1 216 | US$ 0,0056 |
| **Global** | 21 | 21 | 20 | 12 098 | 16 430 | 17 803 | 16 328 | 153 | 13 163 | 1 158 | US$ 0,0398 |

### Observaciones

- **Tiempo: ningún proveedor cumple los 15 s en P95.**
  - La mediana queda por debajo (12,1 s en OpenAI y 13,8 s en Gemini), pero 3 de 21
    generaciones en OpenAI y 4 de 21 en Gemini superan los 15 s. Con 21 muestras se admite
    una sola.
  - Casi todo el tiempo es del proveedor: el sistema suma unos 110 ms, igual que con el
    simulado. Mejorar el P95 depende de lo que se envía y de cómo se pide (riesgo 10): menos
    catálogo por solicitud (hoy unos 13 000 a 16 000 tokens de entrada), otro nivel de
    razonamiento u otro modelo.
  - Si no se alcanza, se documentan las condiciones de la prueba, como prevé el plan de
    contingencia del riesgo.
  - Los tokens de entrada no explican las diferencias entre casos. Los siete envían casi lo
    mismo, y la variación es del proveedor: en OpenAI, el mismo caso C1 tardó entre 15,1 s
    y 17,8 s.
- **Qué explica el tiempo del proveedor: los tokens de salida.** Los tokens de salida
  incluyen el razonamiento del modelo.
  - **OpenAI:** la correlación entre la duración y los tokens de salida es de 0,92. Cada
    token de salida suma unos 10,5 ms, y casi no hay tiempo fijo. Las tres generaciones más
    rápidas produjeron 770 a 994 tokens (8,4 a 10,0 s); las tres más lentas, 1 363 a 1 598
    (15,0 a 17,6 s). A ese ritmo, 15 s equivalen a unos 1 400 tokens de salida.
  - **Gemini:** la relación es más débil (0,59, unos 8 ms por token más 1,2 s fijos) y
    tiene una generación atípica de 20,4 s con 1 542 tokens.
  - **La entrada pesa poco en el tiempo.** Con la entrada casi fija (12 700 a 16 300
    tokens), no se puede medir su efecto en esta corrida. Pero en las muestras del 01/11,
    con 1 769 tokens de entrada, OpenAI tardó lo mismo por token de salida (unos 830
    tokens en 8 a 10 s). Achicar el catálogo baja el costo, pero no el P95.
  - **Consecuencia para el riesgo 10:** lo que mejora el P95 es reducir la salida: un
    razonamiento más bajo, menos opciones por comida o textos más cortos (con una plantilla
    nueva y versionada). Otra opción es repartir la generación en llamadas paralelas por
    comida. Cada cambio se mide con este script contra esta corrida. El razonamiento más
    bajo ya se probó: está en la sección siguiente.
- **Coincide con las muestras del 01/11.** La generación desde el frontend con Lucía tardó
  12 618 ms de proveedor (OpenAI, 13 238 tokens de entrada). Acá la media de OpenAI es de
  12 286 ms, con unos 13 000 tokens.
- **Validez: 42 de 42 salidas reales cumplieron el contrato**, sin `invalid_output` ni
  fallas del proveedor.
- **Bloqueantes: 20 de 21 en los dos proveedores, y la falla es la misma.** En C3, la
  paciente que no cocina y compra comida hecha, los dos modelos propusieron una vez
  comida de rotisería como texto libre: Gemini "Pollo al horno de rotisería" y otros dos,
  OpenAI "Comida hecha de rotisería con una fuente de proteínas, verduras y arroz o papa".
  Quedan como ítems no reconocidos (DA-26). No es un error del modelo sino un hueco del
  catálogo: no hay comida comprada hecha. La entrevista (T35) dirá si hace falta.
- **El 95 % de esta corrida es optimista.** Sin reglas clínicas cargadas, celiaquía,
  diabetes e hipertensión no generan hallazgos. Lo único que puede bloquear son las
  condiciones directas (alergia, intolerancia, rechazo) y los ítems no reconocidos. Con la
  matriz de T35 cargada hay que volver a medir.
- **Condiciones directas: ninguna violada.** C4 no recibió huevo ni recetas con huevo, C5
  no recibió leche y C7 no recibió legumbres. El catálogo enviado ya los excluye (DA-31),
  así que esto mide el filtro, no al modelo.
- **Celiaquía (C1), lectura manual:** los cereales propuestos son siempre las variantes
  "sin TACC" (pan, fideos, galletitas, avena). Gemini usó tres recetas sin trigo, avena,
  cebada ni centeno, y OpenAI ninguna. No apareció la receta de avena común que señaló la
  muestra del 01/11.
- **Composición:** OpenAI dejó 56 advertencias en 21 propuestas y Gemini 20. Las dos son
  advertencias (RF-44), no bloqueos.
- **Costo:** Gemini sale unas 9 veces más caro por generación con los precios actuales, y
  unas 18 veces desde el aumento del 01/01/2027.
- **Muestra chica:** 3 corridas por caso alcanzan para ver el orden de magnitud y comparar
  proveedores, no para comparar casos entre sí. Las diferencias entre casos del mismo
  proveedor (por ejemplo, C1 contra C2 en OpenAI) pueden ser solo variación entre corridas.

## 01/11/2026 — razonamiento más bajo (riesgo 10)

Prueba hecha a partir de la observación anterior: el tiempo del proveedor depende de los
tokens de salida, y esos tokens incluyen el razonamiento. Se repitió la corrida bajando
solo el nivel de razonamiento, con todo lo demás igual: mismos casos, N=3, plantilla
`plan_proposal_v1` y la misma base, con los borradores anteriores todavía cargados. Corrió
de 14:45 a 14:48.

### Resumen para el informe

- **Con el razonamiento en `none`, GPT-6 Luna cumple la meta de tiempo con margen.**
  - El P95 pasó de 16,4 s a 10,5 s, y el máximo de 17,8 s a 10,6 s. Las 21 generaciones
    terminaron en menos de 11 s.
  - Se mantuvieron la validez (21 de 21) y la proporción sin bloqueantes (95,2 %), con el
    mismo costo.
- **Hay señales de menor calidad en el caso de celiaquía.** En 1 de 3 generaciones propuso
  una receta con avena común, y en otra escribió como texto libre una avena que el catálogo
  ya tenía. Con `low` no pasó en ninguno de los 3 intentos. La muestra es chica, pero el
  riesgo es clínico (riesgo 1).
- **Gemini 3.8 Flash no tiene este margen.** No admite un nivel por debajo de `low`, así
  que sigue en un P95 de 17,1 s.

### Niveles probados

| Proveedor | Nivel | Resultado |
|---|---|---|
| OpenAI `gpt-6-luna` | `minimal` | **No admitido.** La API responde `400`: el modelo solo acepta `none`, `low`, `medium` (por defecto), `high`, `xhigh` y `max` ([página del modelo](https://developers.openai.com/api/docs/models/gpt-6-luna)). Las 21 generaciones fallaron en menos de 1 s, sin costo. No se guardó el JSON. |
| OpenAI `gpt-6-luna` | `none` | Medido: [JSON](2026-11-01-openai-gpt-6-luna-razonamiento-none.json) |
| Gemini `gemini-3.8-flash` | `minimal` | **No admitido.** La API responde `400` "Thinking level MINIMAL is not supported for this model". Para este modelo, `low` ya es el mínimo. No se guardó el JSON. |

El `.env.example` de `nutria-ia` lista los niveles que declara cada SDK. Cuáles admite cada
modelo lo decide su API, y esta prueba lo confirma para los dos candidatos.

### OpenAI `gpt-6-luna`: `low` frente a `none`

| | `low` (corrida anterior) | `none` |
|---|---|---|
| Válidas | 21/21 | 21/21 |
| **Total P50** | 12 098 ms | **8 771 ms** |
| **Total P95** (meta ≤ 15 s) | 16 430 ms ✗ | **10 484 ms ✓** |
| Total máximo | 17 803 ms | 10 562 ms |
| Total medio | 12 394 ms | 8 857 ms (−29 %) |
| Proveedor medio | 12 286 ms | 8 753 ms |
| Overhead medio | 108 ms | 104 ms |
| Por encima de 15 s | 3 de 21 | 0 de 21 |
| Tokens de salida (media) | 1 158 | 1 082 (−7 %) |
| Tiempo por token de salida | ≈ 10,6 ms | ≈ 8,1 ms |
| Costo por generación | US$ 0,0019 | US$ 0,0019 |

Tiempos totales de `none`, ordenados (ms):

| Posición | Valores |
|---|---|
| 1.º–10.º | 7 459 · 7 572 · 7 761 · 8 137 · 8 172 · 8 181 · 8 195 · 8 282 · 8 320 · 8 619 |
| **11.º (P50)** | **8 771** |
| 12.º–19.º | 8 810 · 8 935 · 9 085 · 9 146 · 9 551 · 9 968 · 9 987 · 9 992 |
| **20.º (P95)** | **10 484** |
| 21.º (máximo) | 10 562 |

Por caso:

| Caso | Intentos | Válidas | Sin bloqueantes | Total P50 (ms) | Total P95 (ms) | Total máx. (ms) | Proveedor P95 (ms) | Overhead P95 (ms) | Tokens entrada (media) | Tokens salida (media) | Costo |
|---|---|---|---|---|---|---|---|---|---|---|---|
| C1 | 3 | 3 | 2 | 9 992 | 10 562 | 10 562 | 10 477 | 122 | 13 457 | 1 220 | US$ 0,0059 |
| C2 | 3 | 3 | 3 | 8 181 | 8 320 | 8 320 | 8 237 | 83 | 13 476 | 956 | US$ 0,0055 |
| C3 | 3 | 3 | 3 | 8 172 | 9 085 | 9 085 | 8 995 | 148 | 12 690 | 1 017 | US$ 0,0053 |
| C4 | 3 | 3 | 3 | 8 195 | 10 484 | 10 484 | 10 410 | 89 | 13 042 | 1 126 | US$ 0,0056 |
| C5 | 3 | 3 | 3 | 8 771 | 9 551 | 9 551 | 9 388 | 163 | 13 282 | 1 180 | US$ 0,0058 |
| C6 | 3 | 3 | 3 | 9 146 | 9 987 | 9 987 | 9 847 | 140 | 13 444 | 1 073 | US$ 0,0056 |
| C7 | 3 | 3 | 3 | 8 619 | 8 935 | 8 935 | 8 758 | 177 | 12 749 | 1 004 | US$ 0,0053 |
| **Global** | 21 | 21 | 20 | 8 771 | 10 484 | 10 562 | 10 410 | 163 | 13 163 | 1 082 | US$ 0,0390 |

### Calidad: `low` frente a `none`

| | `low` | `none` |
|---|---|---|
| Sin bloqueantes | 20/21 | 20/21 |
| Sin ningún hallazgo | 3/21 | 3/21 |
| Advertencias de composición | 56 | 49 |
| Opciones por propuesta (media) | 14,7 | 15,6 |
| Ítems por propuesta (media) | 24,8 | 22,6 |
| Opciones con receta por propuesta (media) | 10,0 | 12,4 |
| Caracteres de los seis textos (media) | 1 508 | 1 744 |
| Avena común en C1 (celiaquía) | 0 de 3 | 1 de 3 |
| Ítems de texto libre por algo que está en el catálogo | 0 | 2, en un intento de C1 |

Los caracteres de `low` se calcularon después, desde los borradores guardados en la base.
El script todavía no los registraba en esa corrida: desde esta prueba registra las
opciones y el largo de los textos de cada generación.

### Observaciones

- **El ahorro no viene de escribir menos.** Con `none` la salida tiene solo un 7 % menos de
  tokens: los textos son algo más largos y la cantidad de opciones es parecida. Lo que cae
  es el tiempo por token, de ≈ 10,6 ms a ≈ 8,1 ms. Con `low`, el modelo dedica parte del
  tiempo a razonar antes de escribir. La traza guarda la salida y el razonamiento sumados,
  así que no se puede separar cuánto pesa cada uno. Registrarlos aparte (los dos SDK los
  informan) permitiría medirlo. Se registraron después: ver la sección de la plantilla v2.
- **Celiaquía (C1):**
  - **Avena común.** En el intento 2, `none` propuso "Avena cocida con leche y pera", que
    lleva *avena arrollada* común. Es el mismo riesgo que la muestra del 01/11 con
    "Panqueques de avena y banana".
  - **Texto libre.** En el intento 3 escribió "Avena certificada sin TACC" y "Licuado de
    banana con leche y avena certificada sin TACC" como texto libre, aunque el catálogo
    enviado tenía "Avena sin TACC". Quedaron como ítems no reconocidos (el único
    bloqueante de la corrida).
  - **Con `low`** usó "Avena sin TACC" del catálogo en los tres intentos.
  - **El motor no lo detecta todavía.** No hay reglas de celiaquía cargadas.
  - **Con las reglas de T35,** una regla `prohibited` sobre la avena común la saca del
    catálogo enviado, y con ella las recetas que la llevan (DA-31). El riesgo queda
    cubierto por el sistema y no por el modelo.
- **El bloqueo de C3 (rotisería)** no se repitió con `none`. Con 3 intentos por caso eso
  puede ser variación entre corridas.
- **Composición:** 49 advertencias contra 56, dentro de la variación esperable con esta
  muestra.

### Qué queda para decidir

No se cambió la configuración: el `.env` local de `nutria-ia` sigue con `low`, y staging no se tocó. Lo que
muestra la prueba, para DA-12 y para el riesgo 10:

- **OpenAI con `none` es hoy la única configuración medida que cumple las dos metas**, al
  costo más bajo. Conviene adoptarla recién con las reglas de T35 cargadas y volviendo a
  medir con las dos opciones: así el porcentaje sin bloqueantes incluye las patologías y se
  ve si la pérdida de calidad en celiaquía la absorbe el motor.
- **Para mantener `low` y bajar de 15 s,** queda reducir la salida con una plantilla nueva
  (`plan_proposal_v2`, con menos opciones por comida o textos más cortos) o generar las
  comidas en paralelo. Las dos se probaron más abajo: el paralelo sirve para OpenAI pero
  no para el P95 de Gemini, y la plantilla v2 baja el tiempo un 20 % en OpenAI.
- **Registrar aparte los tokens de razonamiento** en la traza, para atribuir el tiempo. Ya
  está hecho: ver la sección de la plantilla v2.

## 01/11/2026 — comidas en paralelo (riesgo 10)

Prueba de la otra vía para bajar el tiempo sin tocar el nivel de razonamiento: en lugar de
una llamada que escribe las cuatro comidas, cuatro llamadas simultáneas de una comida cada
una. Se hizo con los dos proveedores en `low`, el nivel con el que no se llegaba a los 15 s.
Terminó a las 15:20.

### Resumen para el informe

- **GPT-6 Luna:**
  - En paralelo, el P95 bajó de 17,3 s a 10,0 s, y la media de 12,3 s a 7,9 s (−36 %).
    Fue más rápido en 20 de los 21 pares, con un ahorro medio de 4,4 s.
  - El costo por generación se multiplica por 3,2 (de US$ 0,0019 a 0,0062).
  - Las comidas se coordinan peor: hay casi el triple de opciones repetidas entre almuerzo
    y cena o entre desayuno y merienda.
- **Gemini 3.8 Flash:**
  - Bajaron la mediana (de 13,3 s a 9,9 s) y la media (−19 %), pero no el P95 (de 15,9 s a
    16,6 s), y el máximo subió a 21,2 s.
  - Con cuatro llamadas, la espera es la de la más lenta, y Gemini varía mucho entre
    llamadas.
  - El costo por generación sube de US$ 0,018 a 0,057.
- **No se reduce a un cuarto.** Cada llamada vuelve a escribir los seis textos del plan y
  tiene una latencia base. Una comida sola tarda unos 6,5 a 7 s en OpenAI y 8 a 9,5 s en
  Gemini.
- **Comparado con la prueba anterior,** en OpenAI el razonamiento `none` da casi el mismo
  P95 (10,5 s) sin costo extra y sin perder coordinación entre comidas, aunque con el
  riesgo de calidad de celiaquía. El paralelo conserva `low`, pero cuesta el triple y
  requiere cambiar el contrato y el backend.

### Método

- **Directo contra `nutria-ia`** (`POST /plan-proposals`), desde un contenedor del backend
  en la red de Docker. Sin el backend, así que no incluye su overhead de unos 110 ms ni el
  motor de reglas.
- **Contexto de cada caso:** el último `context_sent` exitoso de su paciente en
  `ai_generation`, es decir, el contexto real que arma el backend, con el catálogo completo.
- **Una llamada:** el contexto tal cual, con las cuatro comidas.
- **En paralelo:** cuatro copias del contexto, cada una con `meal_types` de una sola
  comida y solo las recetas candidatas de esa comida, como las filtraría el backend.
  Salen a la vez, y el tiempo es hasta que responde la última.
- **Pares por caso e iteración:** las dos variantes se corrieron una detrás de la otra,
  alternando cuál iba primero, para que la variación del proveedor en el tiempo afecte a
  las dos por igual. N=3, 21 pares por proveedor.
- **Cada llamada sigue pidiendo los seis textos,** porque el contrato los exige. Un plan
  armado así usaría los textos de una sola llamada y descartaría los otros tres.
- **Repeticiones:** opciones de la cena idénticas, por sus ítems, a una del almuerzo, más
  las de la merienda idénticas a una del desayuno. Indican cuánto se coordinan las comidas
  entre sí.
- **El script del experimento no se versionó:** es de una sola vez, y este método alcanza
  para repetirlo. Los resultados, sin el texto de las propuestas, están en
  [OpenAI](2026-11-01-openai-gpt-6-luna-paralelo-por-comida-razonamiento-low.json) y
  [Gemini](2026-11-01-gemini-gemini-3.8-flash-paralelo-por-comida-razonamiento-low.json).

### Tiempos

| | OpenAI, una llamada | OpenAI, en paralelo | Gemini, una llamada | Gemini, en paralelo |
|---|---|---|---|---|
| Válidas | 21/21 | 21/21 | 21/21 | 21/21 |
| **P50** | 11 827 ms | **7 493 ms** | 13 308 ms | **9 850 ms** |
| **P95** (meta ≤ 15 s) | 17 283 ms ✗ | **9 999 ms ✓** | 15 898 ms ✗ | **16 618 ms ✗** |
| Máximo | 18 806 ms | 10 231 ms | 18 327 ms | 21 217 ms |
| Media | 12 344 ms | 7 932 ms (−36 %) | 13 534 ms | 11 011 ms (−19 %) |
| Por encima de 15 s | 2 de 21 | 0 de 21 | 2 de 21 | 2 de 21 |
| Pares en que el paralelo fue más rápido | — | 20 de 21 (−4,4 s de media) | — | 17 de 21 (−2,5 s de media) |

Tiempos ordenados (ms). La mediana es el 11.º valor y el P95 el 20.º:

| | 1.º–10.º | **11.º (P50)** | 12.º–19.º | **20.º (P95)** | 21.º |
|---|---|---|---|---|---|
| OpenAI, una llamada | 9 661 · 9 973 · 10 188 · 10 428 · 10 838 · 11 072 · 11 082 · 11 383 · 11 459 · 11 786 | **11 827** | 12 077 · 12 123 · 12 376 · 12 547 · 13 308 · 13 346 · 13 761 · 13 904 | **17 283** | 18 806 |
| OpenAI, en paralelo | 6 052 · 6 475 · 6 515 · 6 643 · 6 703 · 6 708 · 7 202 · 7 217 · 7 279 · 7 306 | **7 493** | 7 724 · 7 922 · 8 516 · 8 667 · 9 022 · 9 299 · 9 700 · 9 900 | **9 999** | 10 231 |
| Gemini, una llamada | 11 094 · 11 297 · 12 260 · 12 328 · 12 498 · 12 528 · 12 555 · 12 794 · 13 081 · 13 190 | **13 308** | 13 472 · 13 715 · 13 728 · 13 971 · 14 340 · 14 533 · 14 613 · 14 694 | **15 898** | 18 327 |
| Gemini, en paralelo | 8 585 · 8 816 · 8 904 · 8 942 · 8 978 · 9 197 · 9 268 · 9 289 · 9 465 · 9 720 | **9 850** | 10 492 · 10 717 · 10 878 · 10 942 · 11 379 · 11 442 · 12 129 · 14 404 | **16 618** | 21 217 |

Cada comida por separado, en paralelo (media de las 21 llamadas de cada comida):

| Comida | OpenAI: proveedor | OpenAI: tokens de salida | Gemini: proveedor | Gemini: tokens de salida |
|---|---|---|---|---|
| Desayuno | 6 504 ms | 533 | 8 417 ms | 738 |
| Almuerzo | 6 990 ms | 570 | 9 661 ms | 783 |
| Merienda | 6 897 ms | 532 | 8 159 ms | 719 |
| Cena | 6 626 ms | 555 | 9 360 ms | 787 |
| Diferencia media entre la llamada más rápida y la más lenta de una generación | 2,0 s | | 3,5 s | |

### Costo y calidad

| | OpenAI, una llamada | OpenAI, en paralelo | Gemini, una llamada | Gemini, en paralelo |
|---|---|---|---|---|
| Tokens de entrada por generación | 13 163 | 50 647 (×3,8) | 15 900 | 61 269 (×3,9) |
| Tokens de salida por generación | 1 169 | 2 189 (×1,9) | 1 588 | 3 027 (×1,9) |
| **Costo por generación** | US$ 0,0019 | **US$ 0,0062 (×3,2)** | US$ 0,0179 | **US$ 0,0573 (×3,2)** |
| Opciones por plan (media) | 15,3 | 21,4 | 11,7 | 15,7 |
| **Opciones repetidas entre comidas** (media por plan) | 2,4 | **6,3** | 0,05 | **1,8** |
| Planes sin ninguna repetida | 5 de 21 | 1 de 21 | 20 de 21 | 3 de 21 |
| Caracteres de los seis textos | 1 459 | 1 072 (de una de las cuatro llamadas) | 1 619 | 1 256 (ídem) |

La prueba costó unos US$ 1,75: OpenAI 0,17 y Gemini 1,58.

### Observaciones

- **Por qué no baja a un cuarto.** Cada llamada en paralelo escribe los seis textos y una
  comida: 530 a 570 tokens en OpenAI, frente a 1 169 de la llamada única. Además, cada
  llamada tiene una latencia base que no depende del largo. Y como se espera a la más
  lenta de las cuatro, la variación del proveedor pesa más: entre la llamada más rápida y
  la más lenta de una misma generación hay 2 s de media en OpenAI y 3,5 s en Gemini.
- **Gemini no mejora el P95.** Con cuatro llamadas hay cuatro chances de que una sea lenta:
  la máxima fue de 21,2 s, cuando en una llamada fue de 18,3 s. Para Gemini, esta vía baja
  la espera típica pero no la cola, que es lo que mide la meta.
- **El P95 de 21 muestras es inestable.** La llamada única de Gemini dio 15,9 s en esta
  prueba y 17,1 s en la primera corrida del día, a las 14:22 (por el backend, con 0,1 s más
  de overhead). El mismo proveedor se movió más de 1 s en una hora. Las conclusiones valen por la
  comparación en pares, no por el valor absoluto.
- **Coordinación entre comidas.** Cada llamada no sabe lo que proponen las otras, así que
  repite opciones: en OpenAI, cada par de comidas (almuerzo y cena, desayuno y merienda)
  comparte en promedio unas 3 de sus 5 opciones. Además genera más opciones por comida
  (unas 5, contra 3,8 en una llamada), lo que alarga la revisión del profesional.
- **Qué haría falta para implementarlo:**
  - En el contrato, una forma de pedir solo las comidas o solo los textos, para no
    escribir los seis textos cuatro veces. Eso bajaría más el tiempo y el costo, pero no se
    midió.
  - En el backend, unir las cuatro respuestas en un plan.
  - Qué pasa si falla una sola de las cuatro llamadas.
  - Cómo se registra la traza: hoy es una fila de `ai_generation` por plan (DA-27).
  - Cómo se evitan las repeticiones entre comidas, por ejemplo pasándole a cada llamada lo
    que ya propusieron las otras, lo que vuelve a encadenarlas.

  Es un cambio de diseño de DA-28 y DA-31, para una tarea propia.

### Comparación de las vías probadas (OpenAI `gpt-6-luna`)

| | `low`, una llamada | `none`, una llamada | `low`, en paralelo |
|---|---|---|---|
| P95 | 16,4 s – 17,3 s ✗ | 10,5 s ✓ | 10,0 s ✓ |
| Costo por generación | US$ 0,0019 | US$ 0,0019 | US$ 0,0062 |
| Cambios necesarios | Ninguno | Una variable de entorno | Contrato, servicio de IA y backend |
| Riesgo observado | — | Avena común en celiaquía (1 de 3) | Opciones repetidas entre comidas (×2,6) |

El rango de `low` con una llamada son las dos mediciones del día: 16,4 s por el backend a las
14:27 y 17,3 s directo a `nutria-ia` en esta prueba.

## 01/11/2026 — plantilla más corta y tokens de razonamiento (riesgo 10)

Dos cosas en esta prueba:

- **Atribuir el tiempo.** Desde ahora la traza guarda aparte los tokens de razonamiento
  (`ai_generation.reasoning_tokens`, una parte de los de salida). Así se puede ver cuánto
  del tiempo es razonamiento y cuánto es la propuesta.
- **Una plantilla con salida más corta**, `plan_proposal_v2`. Se elige con
  `AI_PLAN_TEMPLATE`, sin desplegar código. La vigente sigue siendo la v1.

Las cuatro corridas fueron por el backend con el script oficial, en `low`, N=3 por caso.
Cada plantilla v2 se midió junto a un **control con la v1 en la misma franja horaria**, de
15:31 a 15:48, porque a lo largo del día los tiempos de la misma configuración se movieron
varios segundos (ver abajo).

### Resumen para el informe

- **Razonamiento:**
  - Con `low`, GPT-6 Luna razona poco: entre 124 y 143 tokens de media, el 11 % al 17 % de
    la salida, y en 7 de cada 21 generaciones no razona nada.
  - A unos 11 ms por token, eso es alrededor de 1,5 s de los 10 a 12 s del proveedor. **El
    tiempo lo explica sobre todo el largo de la propuesta, no el razonamiento.**
  - Gemini 3.8 Flash no informa tokens de razonamiento en estas respuestas.
- **La plantilla v2, con OpenAI:**
  - Pide 2 o 3 opciones por comida y textos de hasta dos oraciones. Contra el control, la
    salida bajó un 21 % (de 1 105 a 868 tokens) y el tiempo un 20 % (media de 12,4 s a
    10,0 s; mediana de 11,8 s a 9,5 s).
  - El P95 pasó de 20,1 s a 13,9 s y cumple la meta, con una sola generación por encima de
    15 s.
  - Mejoraron los indicadores del motor: 7 propuestas sin ningún hallazgo contra 2, y 36
    advertencias de composición contra 46.
  - A cambio, propone menos opciones: 12 por plan contra 15.
- **La plantilla v2, con Gemini:** la salida bajó un 17 %, pero el tiempo solo un 6 % (media
  de 12,9 s a 12,1 s). Gemini tiene más tiempo fijo por llamada, y su P95 depende más de
  las llamadas atípicas que del largo: la v2 tuvo una de 21,4 s.
- **El P95 de 21 muestras no alcanza para decidir entre configuraciones parecidas.** OpenAI
  con `low` y la v1 dio 16,4 s, 17,3 s y 20,1 s en tres momentos del día. Gemini, 17,1 s,
  15,9 s y 14,7 s: esta vez cumplió. Para comparar se usan la media y la mediana en la
  misma franja. La medición definitiva del Incremento 4 necesita más muestras.

### Qué cambia la v2

Es la v1 con dos cambios, para que el modelo escriba menos:

- **Opciones:** "con 2 o 3 opciones intercambiables para cada una". La v1 no fijaba un
  número.
- **Regla 9, nueva:** "Sé breve: cada uno de los seis textos tiene como máximo dos
  oraciones, y la descripción de cada opción es null o un título de hasta cinco palabras."

Todo lo demás es idéntico: las reglas de catálogo, cantidades, contexto fuera del hogar,
contenido clínico y formato. El archivo es `nutria-ia/app/prompts/templates/plan_proposal_v2.txt`.

### Tiempos

| | OpenAI v1 (control) | OpenAI v2 | Gemini v1 (control) | Gemini v2 |
|---|---|---|---|---|
| Archivo | [JSON](2026-11-01-openai-gpt-6-luna-plantilla-v1-control.json) | [JSON](2026-11-01-openai-gpt-6-luna-plantilla-v2.json) | [JSON](2026-11-01-gemini-gemini-3.8-flash-plantilla-v1-control.json) | [JSON](2026-11-01-gemini-gemini-3.8-flash-plantilla-v2.json) |
| Válidas | 21/21 | 21/21 | 21/21 | 21/21 |
| **Media** | 12 444 ms | **9 979 ms (−20 %)** | 12 906 ms | **12 106 ms (−6 %)** |
| **P50** | 11 811 ms | **9 504 ms** | 12 813 ms | **11 997 ms** |
| **P95** (meta ≤ 15 s) | 20 084 ms ✗ | **13 915 ms ✓** | 14 652 ms ✓ | **13 596 ms ✓** |
| Máximo | 20 684 ms | 16 380 ms | 15 528 ms | 21 377 ms |
| Por encima de 15 s | 3 de 21 | 1 de 21 | 1 de 21 | 1 de 21 |
| Tokens de salida (media) | 1 105 | 868 (−21 %) | 1 648 | 1 370 (−17 %) |
| Costo por generación | US$ 0,0019 | US$ 0,0018 | US$ 0,0181 | US$ 0,0171 |

Tiempos totales ordenados (ms). La mediana es el 11.º valor y el P95 el 20.º:

| | 1.º–10.º | **11.º (P50)** | 12.º–19.º | **20.º (P95)** | 21.º |
|---|---|---|---|---|---|
| OpenAI v1 (control) | 9 114 · 9 250 · 9 341 · 9 687 · 9 990 · 10 487 · 10 742 · 11 052 · 11 754 · 11 754 | **11 811** | 12 095 · 12 617 · 12 672 · 12 829 · 12 898 · 13 025 · 13 566 · 15 870 | **20 084** | 20 684 |
| OpenAI v2 | 6 157 · 6 935 · 7 164 · 7 529 · 7 583 · 7 676 · 7 768 · 8 211 · 8 877 · 9 348 | **9 504** | 9 722 · 9 945 · 10 078 · 11 374 · 11 549 · 12 897 · 13 452 · 13 501 | **13 915** | 16 380 |
| Gemini v1 (control) | 10 035 · 10 993 · 11 195 · 11 589 · 11 700 · 12 260 · 12 299 · 12 532 · 12 554 · 12 592 | **12 813** | 12 858 · 13 092 · 13 359 · 13 862 · 13 891 · 14 267 · 14 416 · 14 549 | **14 652** | 15 528 |
| Gemini v2 | 8 454 · 8 755 · 10 442 · 10 688 · 11 132 · 11 588 · 11 592 · 11 780 · 11 781 · 11 919 | **11 997** | 12 007 · 12 117 · 12 191 · 12 205 · 12 476 · 12 544 · 12 704 · 12 888 | **13 596** | 21 377 |

### Atribución del tiempo (OpenAI `gpt-6-luna`, `low`)

| | v1 (control) | v2 |
|---|---|---|
| Tokens de salida (media) | 1 105 | 868 |
| De razonamiento (media) | 124 (11 %) | 143 (17 %) |
| De la propuesta (media) | 982 | 725 |
| Generaciones sin razonamiento | 7 de 21 | 7 de 21 |
| Máximo de razonamiento | 343 | 386 |
| Tiempo por token de salida | ≈ 11,2 ms | ≈ 11,4 ms |
| Correlación tiempo ↔ tokens de salida | 0,93 | 0,95 |
| Las 3 más rápidas (proveedor / salida / razonamiento) | 9,0 s / 848 / 0 · 9,1 s / 853 / 199 · 9,3 s / 938 / 0 | 6,1 s / 578 / 0 · 6,9 s / 638 / 0 · 7,1 s / 673 / 0 |
| Las 3 más lentas | 15,8 s / 1 573 / 191 · 20,0 s / 1 789 / 343 · 20,5 s / 1 497 / 285 | 13,4 s / 1 211 / 323 · 13,8 s / 1 206 / 386 · 16,1 s / 1 632 / 244 |

- **El tiempo por token es el mismo con las dos plantillas,** unos 11 ms. La v2 ahorra en
  proporción a lo que acorta.
- **El razonamiento suma, pero poco.** Las generaciones más lentas razonan más (240 a 390
  tokens) y además escriben más. Con unos 130 tokens de media, el razonamiento explica
  alrededor de 1,5 s.
- **Lo que no explica es el `none`.** Con `none`, OpenAI escribió 1 082 tokens en 8,8 s de
  media, unos 8,1 ms por token. Si la diferencia con `low` fuera solo el razonamiento, a esa
  velocidad los 1 105 tokens de `low` con la v1 tardarían unos 9 s, no 12,4 s. El modo sin
  razonamiento también genera más rápido cada token. Es una observación de dos corridas a horas distintas, no
  una medición controlada.
- **Gemini:** `usage_metadata.thoughts_token_count` vino vacío en las 42 respuestas. La API
  de Gemini omite los campos que valen cero, así que lo más probable es que con `low` no
  razone, pero no se puede confirmar. Se registra como no informado (`null`), no como cero.
  Su tiempo depende menos del largo (correlación de 0,52 a 0,77) y tiene más tiempo fijo
  por llamada.

### Calidad

| | OpenAI v1 (control) | OpenAI v2 | Gemini v1 (control) | Gemini v2 |
|---|---|---|---|---|
| Sin bloqueantes | 20/21 | 20/21 | 20/21 | 21/21 |
| Sin ningún hallazgo | 2/21 | 7/21 | 11/21 | 12/21 |
| Advertencias de composición | 46 | 36 | 21 | 15 |
| Opciones por plan (media) | 14,8 | 12,0 | 12,3 | 11,4 |
| Ítems por plan (media) | 23,4 | 18,0 | 26,9 | 22,6 |
| Caracteres de los seis textos (media) | 1 446 | 953 (−34 %) | 1 637 | 1 152 (−30 %) |

- **Los bloqueos son del mismo tipo que en las corridas anteriores:** comidas fuera de casa
  sin control del paciente, propuestas como texto libre.
  - OpenAI v1, C3: "Comida hecha con una fuente de proteína, verduras y un acompañamiento".
  - OpenAI v2, C7: "Una preparación del menú del comedor escolar…".
  - Gemini v1, C3: "Pollo al spiedo o al horno sin piel de rotisería".

  Es el mismo hueco del catálogo para la entrevista (T35).
- **Menos opciones también es menos variedad para el paciente.** La v2 propone 3 por comida
  en lugar de 3 a 4 en OpenAI. Si alcanza es una decisión profesional para la entrevista.
- **Textos un 30 % más cortos.** Conviene que la nutricionista revise si dos oraciones
  alcanzan para el diagnóstico situacional y las recomendaciones, que después edita.

### El mismo P95 a lo largo del día

| Configuración | 14:22–14:32 (por el backend) | 15:05–15:20 (directo a `nutria-ia`) | 15:31–15:48 (por el backend) |
|---|---|---|---|
| OpenAI `low`, v1 | 16 430 ms | 17 283 ms | 20 084 ms |
| Gemini `low`, v1 | 17 132 ms | 15 898 ms | 14 652 ms |

Con 21 muestras, el P95 es la segunda generación más lenta, y una sola llamada atípica del
proveedor lo mueve varios segundos. Por eso los efectos de esta prueba se leen en la media
y la mediana contra un control de la misma franja, y no en si el P95 cruzó los 15 s en una
corrida.

### Comparación de todas las vías probadas (OpenAI `gpt-6-luna`)

| | `low`, v1 | `low`, v2 | `none`, v1 | `low`, en paralelo |
|---|---|---|---|---|
| Media | 12,3 – 12,4 s | 10,0 s | 8,9 s | 7,9 s |
| P95 | 16,4 – 20,1 s ✗ | 13,9 s ✓ | 10,5 s ✓ | 10,0 s ✓ |
| Costo por generación | US$ 0,0019 | US$ 0,0018 | US$ 0,0019 | US$ 0,0062 |
| Cambio necesario | Ninguno | `AI_PLAN_TEMPLATE` (ya implementado) | `AI_REASONING_LEVEL` | Contrato, servicio de IA y backend |
| Riesgo observado | — | Menos opciones y textos más cortos | Avena común en celiaquía (1 de 3) | Opciones repetidas entre comidas |

`none` con la v2 no se probó. Por lo medido, sería la combinación más rápida (las dos
reducciones se suman), con el riesgo de calidad de `none`.

### Qué queda para decidir

No se cambió la configuración: el `.env` local de `nutria-ia` sigue con `low` y la v1, y
staging no se tocó.

- **La v2 es la vía más barata para cumplir la meta sin tocar el razonamiento:** una
  variable de entorno, sin costo extra, y con mejores indicadores del motor. La decisión
  es de contenido: si 3 opciones por comida y textos de dos oraciones le alcanzan al
  profesional. Conviene mostrarla en la entrevista (T35) junto a una propuesta de la v1.
- **La medición definitiva** (Incremento 4) necesita más muestras por configuración, por
  ejemplo N=10 por caso (70 generaciones), y comparar en la misma franja horaria.

## Cómo comparar

Para que una corrida nueva sea comparable con estas:

- Mismo `N`, mismos casos, mismo modelo, mismo razonamiento y la misma plantilla, o la
  diferencia anotada en las condiciones.
- Una sección nueva en este archivo con la misma tabla de condiciones, el resumen por
  proveedor y las tablas que imprime el script, más el JSON en esta carpeta.
- **Con RAG:** el tiempo total va a sumar la recuperación antes de la llamada, y el
  overhead es la columna que lo muestra. Comparar también con las
  [muestras sin RAG](../muestras-de-generacion/README.md).
- **Con las reglas de T35 cargadas:** el porcentaje sin bloqueantes pasa a medir las
  patologías. No mezclar ese número con el de esta corrida.
