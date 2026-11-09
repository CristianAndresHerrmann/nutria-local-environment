# Catálogo base de alimentos

Origen, criterios y pendientes del catálogo común de alimentos que carga el seed (T28;
RF-24, RF-25; DA-21 y DA-32). La lista completa vive en
`nutria-backend/scripts/seed-data/catalogs.js` y no se repite acá.

## Volumen y origen

| | |
|---|---|
| Grupos raíz | 12 |
| Subgrupos | 32 |
| Alimentos comunes | 232 (el objetivo de DA-21 es de 150 a 250) |
| Con categoría `complementary` | 38 |
| Origen | **V** (pendiente de validación) en todo el catálogo |

Es una propuesta del desarrollador a partir de los grupos de las *Guías Alimentarias para
la Población Argentina* (GAPA, Ministerio de Salud de la Nación, adoptadas por la
Resolución 693/2019), la misma fuente de DA-29. No la revisó una nutricionista. Las 31
asignaciones del seed anterior se conservan y pasan al mismo régimen. Las categorías de
cada alimento se llevan a la entrevista de validación, porque las advertencias de
composición de T27 dependen de ellas.

El relevamiento original lo armó el desarrollador con apoyo de un asistente de IA. Quien
lo cargó es responsable de su contenido: la IA no es un origen.

## Criterios

- **Materia prima, no preparaciones (DA-21).** Un alimento se consume solo o como
  ingrediente. Milanesas, tartas o ñoquis son recetas (T30). Se incluyen productos
  envasados de uso habitual como ingrediente (tapas de tarta, pan lactal, galletitas).
- **Dos niveles de grupos.** Los alimentos van siempre en un grupo hoja; un grupo raíz sin
  subgrupos (Infusiones, Bebidas) es hoja.
- **Taxonomía neutra (DA-21).** Los subgrupos se arman por origen o uso culinario, no por
  restricción clínica. Las variantes sin TACC están en su subgrupo natural (pan sin TACC
  en Panificados y harinas). Qué alimentos restringe cada patología lo define el motor de
  reglas, con su ficha en `reglas-clinicas.md`.
- **Categorías (RF-25)**, con las convenciones del seed: lácteos como proteína, papa como
  hidrato, frutas como fibra (ninguna como hidrato: la fruta no reemplaza al hidrato del
  plato), legumbres secas y en conserva como proteína, hidrato y fibra, palta como grasa
  saludable. Alimentos sin aporte a la composición: `complementary` (DA-32). El ajo y el
  queso rallado conservan `vegetable` y `protein` aunque se usen como condimento. Si no,
  los ítems de grupo Lácteos, Quesos y "Bulbos, raíces y tallos" se quedan sin categoría,
  porque un grupo aporta solo las que comparten todos sus alimentos (DA-30, DA-42).
- **Productos envasados sin "exceso en sodio".** Panes, quesos, conservas, aceitunas y demás
  envasados del catálogo se consideran versiones sin el sello "exceso en sodio". Es una
  premisa del proyecto (DA-43), coherente con la GPC HTA, p. 14: "preferir los que no estén
  rotulados como 'exceso en sodio'". Por eso no tienen regla en hipertensión. Las reglas
  cubren lo que es alto en sodio por naturaleza, con o sin sello: sal fina, caldo en cubo,
  salsa de soja, fiambres y embutidos. La recomendación de Hipertensión le indica al
  paciente que elija esas versiones.
- **Compatibilidad con el seed.** El seed hace upsert por nombre y no borra. Los 15 nombres
  de grupo y los 31 de alimento anteriores no se renombran, porque los grupos no tienen
  baja lógica y las condiciones y recetas del seed los referencian por nombre. Algunos
  alimentos cambian de subgrupo (Leche a Leches y yogures, Fideos secos a Pastas).
- **Kéfir** no está en el catálogo común: es el alimento privado de ejemplo del seed, y el
  servicio rechaza un nombre común repetido.

## Cobertura de los casos de prueba

El catálogo da alimentos identificables y alternativas para las tres patologías de los
casos de prueba. No define reglas: eso va en `clinical_rule`.

| Patología | Qué aporta el catálogo |
|---|---|
| Celiaquía | Variantes sin TACC separadas de sus equivalentes con trigo, avena, cebada o centeno (pan, premezcla, fideos, galletitas, avena, copos de maíz), y harinas de arroz, maíz, mandioca y garbanzo. Los copos de maíz comunes suelen llevar malta de cebada (DA-42). |
| Diabetes | Azúcares y dulces en un grupo propio, con edulcorante y mermelada reducida en azúcar como alternativas, y versiones integrales de pan, fideos, arroz y galletitas. |
| Hipertensión | Sal, caldo en cubo, salsa de soja, fiambres y embutidos como alimentos identificables. La sal con bajo contenido de sodio está en el catálogo, pero no es una alternativa recomendada: es cloruro de potasio, y su uso queda para la entrevista (DA-42). |

## Verificación

`nutria-backend/scripts/seed-data/catalogs.test.js` controla los datos sin base: nombres
únicos sin distinguir mayúsculas (alimentos y grupos), grupos existentes y hoja, padres
declarados antes que sus hijos, al menos una categoría válida y sin repetir, y que los
nombres que el seed referencia sigan existiendo. También las correcciones de DA-42:
ninguna fruta con `carbohydrate` y las legumbres secas con `carbohydrate`.
`clinical-cases.rules.test.js` corre el motor con estas categorías y controla que los
ítems de grupo (un lácteo, un queso, un bulbo, una legumbre) completen la opción.

## Pendientes para la entrevista de validación

- [ ] Revisar el listado: alimentos que faltan, sobran o se nombran distinto en la consulta.
- [ ] Confirmar el nivel de detalle de las carnes (cortes individuales o "carne vacuna magra").
- [ ] Confirmar si se incluyen bebidas azucaradas y gaseosas, hoy excluidas.
- [ ] Validar las categorías asignadas, junto con los puntos abiertos de DA-29 (fruta y lácteo).
