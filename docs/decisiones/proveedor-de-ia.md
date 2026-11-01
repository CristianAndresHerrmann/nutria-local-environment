# Proveedor de IA: GPT-6 Luna por defecto, Gemini 3.8 Flash como alternativa (DA-12)

**Fecha:** 01/11/2026
**Tarea relacionada:** DA-12 (depende de T36, T37 y T41)

## Contexto

La evaluación preliminar del IA2 (Tabla 6) dejó dos candidatos principales: GPT-6 Luna
(OpenAI) y Gemini 3.8 Flash (Google). Con los dos adaptadores implementados detrás de la
misma interfaz (T36, [ficha](adaptadores-de-proveedores-de-ia.md)), el benchmark propio se
corrió el 01/11/2026 con el script de T41. Se usaron los 7 casos ficticios del seed, la
misma plantilla y el mismo esquema de salida, por el recorrido completo del backend y con
la matriz de reglas de T35 cargada. Resultados y método en
[`mediciones-de-generacion/`](../mediciones-de-generacion/README.md), sección "Benchmark
DA-12"; las corridas anteriores del mismo día, sin reglas de patología, están en el mismo
archivo.

Criterios del IA2 ("Criterios de selección del proveedor de IA") y lo que dio cada uno.
Salvo que se indique otra cosa, los valores son de razonamiento `low`, con N=3 por caso:

| Criterio | GPT-6 Luna | Gemini 3.8 Flash |
|---|---|---|
| **Tiempo total P95** (meta 15 s), plantilla v1 | 16,4 – 20,1 s en cuatro corridas: no cumple | 14,7 – 17,1 s en cuatro corridas: cumple una |
| **Tiempo total P95**, plantilla v2 | 13,9 s y 11,1 s: cumple | 13,6 s y 15,6 s (máximo 22,9 s): cumple una |
| Mediana con la v1 | 11,8 – 14,3 s | 11,9 – 13,8 s |
| Cola (generaciones de una llamada de más de 20 s, `low`) | 2 en 84 con la v1 (20,1 y 20,7 s), ninguna en 42 con la v2 | 3 en 126: 20,6 s con la v1; 21,4 y 22,9 s con la v2 |
| Nivel de razonamiento más bajo | `none`: P95 de 10,5 s y 11,8 s | `low` (`minimal` no lo admite) |
| **Sin bloqueantes** con la matriz de T35 (meta 95 %) | 95,2 % con la v1 y con la v2 | 100 % con la v1; 85,7 % con la v2 |
| Bloqueos por reglas de patología | Ninguno | Ninguno |
| **Salida estructurada** | `json_schema` estricto; 0 fallas en 147 generaciones por el backend, con niveles de razonamiento admitidos | `response_json_schema`, sin `maxItems` (T36); 1 `provider_error` en 105, ninguna salida inválida |
| **Consistencia** (riesgo 19) | CV del tiempo de 6 % a 12 %; repite cerca de la mitad de las opciones entre corridas | CV de 6 % a 9 %; mismos alimentos, opciones casi siempre distintas |
| **Español** | Voseo correcto; formulaico, con referencias al contexto recibido y errores de tilde menores | Voseo correcto; más natural y concreto; diagnóstico en tercera persona con la v1 |
| Composición (advertencias en 21 propuestas, con T35) | 55 con la v1, 22 con la v2 | 16 con la v1, 12 con la v2 |
| **Costo por generación** | US$ 0,0017 – 0,0019 | US$ 0,016 – 0,018; el doble desde el 01/01/2027 |
| TTFT | No se midió: sin streaming, y la propuesta se valida completa | Ídem |
| **Política de datos** (T36) | Sin entrenamiento salvo opt-in; 30 días de monitoreo de abusos; `store=false`; retención cero y residencia de datos con aprobación de ventas | Sin entrenamiento en el tier pago; retención "por un período limitado", sin retención cero; los términos prohíben usarlo "in clinical practice" o "to provide medical advice" |

## Alternativas consideradas

- **Gemini 3.8 Flash por defecto.** Es el que mejor escribe: textos más concretos y
  naturales, menos advertencias de composición y el único 100 % sin bloqueantes (con la
  v1).
  - En contra, no cumple de forma estable la meta de tiempo con ninguna plantilla. Su
    mínimo de razonamiento es `low`, y tiene la cola más larga: lo que la meta mide.
  - Cuesta unas 9 veces más por generación, y 18 veces desde el 01/01/2027.
  - Sus términos prohíben el uso "in clinical practice", algo que habría que resolver antes
    de usarlo en un sistema para nutricionistas.
  - El 100 % de la v1 no se repitió con la v2 (85,7 %, con una falla del proveedor).
- **GPT-6 Luna con razonamiento `none`.** Es la más rápida con la v1 (P95 de 10,5 s y
  11,8 s) y no exige cambiar la plantilla.
  - Pero baja la calidad. Con la matriz de T35 quedó en 85,7 % sin bloqueantes: adaptó como
    texto libre recetas que el catálogo había excluido ("panqueques… preparados sin leche"
    a una paciente intolerante a la leche).
  - Antes de T35 había propuesto avena común en celiaquía.
- **No decidir hasta el Incremento 4**, con N=10 por caso. Daría un P95 más firme, pero el
  Incremento 3 necesita un proveedor por defecto para staging. Las diferencias de costo,
  política de datos y cola no dependen del tamaño de la muestra.
- **Elegir por la Tabla 6 sin benchmark.** Es lo que el IA2 descartó: la Tabla 6 sirvió
  para reducir candidatos, no para decidir.

## Decisión

- **GPT-6 Luna (`gpt-6-luna`) es el proveedor por defecto,** con razonamiento `low`:
  `AI_PROVIDER=openai`, `AI_MODEL=gpt-6-luna` y `AI_REASONING_LEVEL=low` en el `.env` de
  `nutria-ia`, en local y en staging. El código conserva `fake` como valor por defecto
  para que la CI y los tests no necesiten claves (T33).
- **Gemini 3.8 Flash queda como alternativa,** con su adaptador, sus tests y su
  configuración documentada. Cambiar de proveedor es cambiar dos variables y recrear el
  servicio, sin desplegar código (riesgo 5).
- **No se usa `none`.** La pérdida de calidad se vio en dos corridas, y en un caso se le
  escapa al motor (texto libre).
- **La plantilla no la fija esta decisión.** Con la v1 vigente, el P95 no se cumple con
  ningún proveedor. GPT-6 Luna con la v2 lo cumplió en las dos corridas, con mejores
  indicadores del motor. Adoptar la v2 es una decisión de contenido: si 3 opciones por
  comida y textos de dos oraciones le alcanzan al profesional. Queda como recomendación
  para la revisión con una nutricionista (ver
  [la sección de la v2](../mediciones-de-generacion/README.md#01112026--plantilla-más-corta-y-tokens-de-razonamiento-riesgo-10)).

Por qué GPT-6 Luna:

- **Tiempo.** Es el único candidato que cumple la meta de forma repetida con una
  configuración disponible hoy (la v2), y su cola es más corta.
- **Calidad dentro de la meta.** Cumple el 95 % sin bloqueantes con las dos plantillas, y
  sus bloqueos son el mismo hueco de catálogo que tiene Gemini.
- **Costo.** Es un orden de magnitud más barato, compatible con el carácter académico del
  proyecto (criterio de costo del IA2).
- **Política de datos.** No restringe el uso en salud, y si hiciera falta, ofrece retención
  cero y residencia de datos.
- **Lo que pierde contra Gemini** (redacción, variedad entre corridas, composición) se
  corrige en la plantilla o lo marca el motor como advertencia. No es un riesgo clínico.

## Consecuencias

- **Staging.** El `.env` de `nutria-ia` en la VM lleva `AI_PROVIDER=openai`,
  `AI_MODEL=gpt-6-luna` y `OPENAI_API_KEY`. La clave de Gemini es opcional allí. El compose
  no cambia.
- **Costo de operación.** Unos US$ 0,002 por generación con los precios del 01/11/2026,
  unos US$ 2 cada mil generaciones. Ya no queda cubierto por el crédito de Google AI Pro
  que preveía `AGENTS.md`.
- **Mejoras de plantilla pendientes**, independientes del proveedor y para una versión
  nueva:
  - el diagnóstico en segunda persona;
  - no mencionar el contexto recibido;
  - "tu nutricionista" en lugar de "la persona nutricionista";
  - no adaptar como texto libre una receta que no está en el catálogo.
- **Hueco del catálogo.** Los bloqueos que quedan son comidas fuera de casa sin control del
  paciente (rotisería, comedor escolar), con los dos proveedores. Lo resuelve un ítem de
  catálogo, a revisar con una nutricionista.
- **Medición definitiva** en el Incremento 4 (Tabla 7), con el proveedor elegido, más
  muestras (N=10 por caso) y la plantilla que se adopte.
- **Riesgos:**
  - Riesgo 5: mitigado por el adaptador alternativo probado con el mismo benchmark.
  - Riesgo 10: la meta se cumple con la v2, pendiente de adoptar.
  - Riesgo 18: el contenido clínico en texto libre es parecido en los dos proveedores,
    mitigado por la plantilla y la revisión profesional.
  - Riesgo 19: la variabilidad queda medida y no afecta la validez.
- **Si cambia el precio o la disponibilidad de GPT-6 Luna,** se vuelve a correr el
  benchmark con `npm run measure:generation` en la misma franja horaria, y Gemini entra
  cambiando la configuración. Antes de usar Gemini en producción hay que resolver su
  restricción de uso clínico.
