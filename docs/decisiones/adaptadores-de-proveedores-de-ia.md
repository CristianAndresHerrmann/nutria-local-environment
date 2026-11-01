# Adaptadores de Gemini y OpenAI detrás de la interfaz de proveedor

**Fecha:** 01/11/2026
**Tarea relacionada:** T36

## Contexto

DA-12 elige el proveedor de IA entre los dos candidatos principales, Gemini 3.8 Flash y
GPT-6 Luna. Para compararlos con el mismo contrato hace falta un adaptador real de cada
uno detrás de la interfaz de T33 (DA-28), sin tocar el contrato, el servicio ni la
plantilla. Hubo que resolver lo siguiente:

- Qué API de cada proveedor usar.
- Cómo pedir salida estructurada sin mantener un segundo esquema a mano.
- Cómo se configuran el proveedor, el modelo, el razonamiento y las claves (RNF-06).
- Qué datos guarda el proveedor y cómo se reduce eso (RNF-07).
- Cómo se prueba sin llamadas reales ni claves en la CI.

## Alternativas consideradas

- **Gemini: Interactions API en vez de `generateContent`.** Es la que recomienda Google
  para desarrollos nuevos, y la documentación de Gemini 3.8 Flash solo la usa en sus
  ejemplos. En contra, guarda la interacción del lado del proveedor por defecto
  (`store=true`) para la conversación con estado, que acá no se usa.
  `generateContent` "sigue totalmente soportado" y no tiene estado.
- **OpenAI: Chat Completions en vez de Responses API.** Las dos admiten GPT-6 Luna y
  salida estructurada. Responses es la recomendada, y su `store` se puede apagar.
- **Llamar por HTTP con `httpx` en vez de usar los SDK.** Evita dos dependencias, pero
  obliga a mantener el formato de request y de respuesta y el mapeo de errores de cada
  API. Los SDK suman unos 20 paquetes y la imagen queda en 208 MB, lejos del límite de
  0,5 GB.
- **Escribir el JSON Schema a mano.** Se podría ajustar a cada proveedor, pero sería un
  segundo contrato que se desincroniza del de Pydantic sin que ningún test lo note.
- **Una sola variable `AI_PROVIDER_API_KEY`.** Es más simple, pero comparar los dos
  proveedores obligaría a cambiar también la clave en cada cambio.
- **Tests contra la API con una clave de prueba en un secret.** Prueban la integración
  real, pero la CI no usa secrets (T12) y cada corrida tendría costo.

## Decisión

### Configuración

`AI_PROVIDER=gemini|openai`, con `AI_MODEL` (sin default en el código), `AI_REASONING_LEVEL`
(`low` por defecto) y una clave por proveedor (`GEMINI_API_KEY`, `OPENAI_API_KEY`), solo
en `.env`. Si falta el modelo o la clave del proveedor elegido, o el nivel de
razonamiento no está entre los que declara el SDK, el servicio no arranca. Las claves
quedan fuera del `repr` de la configuración.

### Llamada a cada proveedor

| | Gemini | OpenAI |
|---|---|---|
| API | `generateContent` (`google-genai`) | Responses API (`openai`) |
| Salida estructurada | `response_mime_type=application/json` + `response_json_schema` | `text.format` `json_schema` con `strict: true` |
| Razonamiento | `thinking_level` | `reasoning.effort` |
| Almacenamiento | `store=false`: no registra la solicitud aunque el proyecto lo tenga activado | `store=false`: sin estado de aplicación (30 días por defecto) |
| Reintentos del SDK | Ninguno (`attempts=1`) | Ninguno (`max_retries=0`) |
| Tokens de salida | Respuesta + razonamiento | `output_tokens` (ya incluye el razonamiento) |

Al proveedor viaja solo el prompt renderizado, nunca la solicitud. Ningún SDK reintenta,
porque `AI_PROVIDER_TIMEOUT_SECONDS` cubre la generación completa y el reintento lo
decide el profesional (RNF-02).

Los errores del SDK se traducen a los errores controlados de DA-28:

| Situación | Error |
|---|---|
| Timeout del cliente | `PROVIDER_TIMEOUT` |
| Error HTTP, de conexión, rechazo del modelo o respuesta sin texto | `PROVIDER_ERROR` |
| Texto que no cumple el contrato, incluida una respuesta truncada | `INVALID_PROVIDER_OUTPUT`, con la respuesta cruda |

### Esquema de salida derivado del contrato

`app/providers/schema.py` genera el JSON Schema a partir de `PlanProposal` y lo reduce al
subconjunto que aceptan los dos en modo estricto:

- Sin `$ref`, y las uniones como `anyOf`.
- Todas las propiedades requeridas; las opcionales admiten `null`.
- Sin títulos ni descripciones, así lo único que instruye al modelo es la plantilla
  versionada.

Se descartan los largos de texto y el patrón de los códigos, que Gemini no aplica. Al
esquema de Gemini se le quitan además los `maxItems`: con los límites anidados del
contrato (comidas × opciones × ítems) la API responde `400 INVALID_ARGUMENT`, y sin
ellos acepta el mismo esquema. Todo lo descartado lo sigue exigiendo la validación
estricta del servicio: el esquema guía al modelo, no reemplaza al contrato.

### Pruebas

Cada adaptador recibe el cliente del SDK por constructor. Los tests pasan uno simulado
que registra la llamada y devuelve respuestas reales del SDK (o levanta sus errores), y
verifican lo que se envía, el mapeo de tokens y cada error controlado. Otro test
comprueba que el esquema cumple el modo estricto y que una propuesta válida lo cumple.
La CI no hace llamadas reales ni necesita claves.

## Política de datos de cada API (insumo de DA-12)

Relevada el 01/11/2026 de las páginas oficiales. No es una interpretación legal.

| | Gemini API (tier pago) | OpenAI API |
|---|---|---|
| Uso para entrenamiento | No usa prompts ni respuestas para mejorar sus productos; actúa como encargado de tratamiento (Data Processing Addendum). En el tier gratuito sí los usa, con posible revisión humana. | No, salvo opt-in explícito. |
| Retención | Registro de prompts y respuestas "por un período limitado", solo para detectar abusos y por obligaciones legales. Los términos actuales no dan la cifra; un aviso de Google de noviembre de 2024 hablaba de 55 días. | Registros de monitoreo de abusos hasta 30 días. Estado de aplicación de la Responses API: 30 días con `store=true`; ninguno con `store=false`. |
| Retención cero | No disponible en Gemini API (Vertex AI la ofrece en algunos endpoints). | Zero Data Retention o Modified Abuse Monitoring, con aprobación de OpenAI (ventas). |
| Jurisdicción y ubicación | Los datos "pueden almacenarse transitoriamente o en caché en cualquier país" donde Google tenga instalaciones. Sin región elegible. | Proyecto global por defecto. Residencia de datos por proyecto, con aprobación de ventas: procesamiento regional en EE. UU., Europa y EAU; solo almacenamiento en otras regiones, entre ellas ninguna de América Latina. Recargo del 10 %. |
| Uso en salud | Los términos prohíben usar el servicio "in clinical practice" o "to provide medical advice". | Sin restricción equivalente en la guía de datos (las políticas de uso no se revisaron en esta tarea). |

Consecuencias para DA-12:

- **La clave de Gemini tiene que ser de un proyecto con facturación activa.** Con una
  clave del tier gratuito, los datos se usan para entrenamiento, y eso choca con RNF-07
  aunque el contexto esté anonimizado.
- **Ninguno de los dos procesa en Argentina.** Si se exige residencia, solo OpenAI ofrece
  una región, y con un trámite comercial.
- **La restricción de uso clínico de Gemini** hay que revisarla contra el alcance de
  NutrIA (una propuesta que revisa y aprueba el profesional, sin indicaciones clínicas
  generadas por la IA, DA-28) antes de elegirlo.

Fuentes:
[Términos de Gemini API](https://ai.google.dev/gemini-api/terms) (última
actualización 28/04/2026);
[Migración a Interactions API](https://ai.google.dev/gemini-api/docs/migrate-to-interactions);
[Datos en la API de OpenAI](https://developers.openai.com/api/docs/guides/your-data);
[Modelo GPT-6 Luna](https://developers.openai.com/api/docs/models/gpt-6-luna).

## Consecuencias

- **Comparar proveedores** es cambiar `AI_PROVIDER` y `AI_MODEL`, sin desplegar código
  (riesgo 5). La traza de cada generación ya informa proveedor, modelo, duración y
  tokens, que son los datos del benchmark.
- **Un cambio en el contrato** se refleja solo en el esquema que se envía. Si un
  proveedor deja de aceptarlo, la generación falla con `PROVIDER_ERROR` y el log
  muestra el status.
- **Gemini puede devolver más ítems u opciones de los que admite el contrato**, porque
  su esquema no lleva `maxItems`. Esa respuesta sale como `INVALID_PROVIDER_OUTPUT` y
  cuenta en el indicador de propuestas rechazadas.
- **Verificación con los proveedores reales.** El caso de T41 se reemplaza por el
  ejemplo del contrato (celiaquía, datos ficticios) hasta que T41 exista. Hay que
  repetirlo con un caso de T41. Una corrida por proveedor el 01/11/2026, con
  razonamiento `low`, `POST /plan-proposals` directo al servicio:

  | Proveedor | Modelo | Resultado | Duración | Tokens de entrada | Tokens de salida |
  |---|---|---|---|---|---|
  | OpenAI | `gpt-6-luna` | 200, cumple el contrato | 9 870 ms | 1 769 | 845 |
  | Gemini | `gemini-3.8-flash` | 200, cumple el contrato | 6 727 ms | 1 677 | 1 107 |

  Es una sola muestra con el catálogo de ejemplo (8 alimentos), no el benchmark de
  DA-12: con el catálogo completo de 231 alimentos la entrada crece y la latencia
  también. Las respuestas, la solicitud y las corridas repetidas están en
  [`muestras-de-generacion/`](../muestras-de-generacion/README.md), como línea de base
  para comparar cuando se implemente RAG.
- **Staging.** Las claves van en el `.env` de `nutria-ia` en la VM. El compose no cambia.
