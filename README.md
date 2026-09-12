actual

- [x] Módulo 1 — Configuración y logging
- [x] Módulo 2 — Loaders
- [x] Módulo 3 — Text Extractor
- [x] Módulo 4 — Document Analyzer estructural
- [x] Módulo 5 — Financial Candidate Finder
- [ ] Capa del único LLM
- [ ] Financial Fact Candidates
- [ ] Financial Parser
- [ ] Consensus Comparison
- [ ] Score Engine
- [ ] Reports / Decision
- [ ] Backtesting
- [ ] Desktop GUI
- [ ] Live / automatización

---

## Arquitectura vigente

```text
Earnings Report
  ↓
Loader
  ↓
Text Extractor
  ↓
Document Analyzer
  ↓
Financial Candidate Finder
  ↓
Única ejecución del LLM
  ↓
Financial Fact Candidates
  ↓
Financial Parser
  ↓
Consensus / otras etapas cuantitativas
  ↓
Análisis cualitativo y combinación de señales
  ↓
Reports / Decision
```

El proyecto utiliza un único LLM. La lógica determinista y la lógica de
interpretación permanecen separadas:

```text
Document Analyzer
= estructura documental y headings linealizados

Financial Candidate Finder
= búsqueda literal y preparación de evidencia

Único LLM
= comprensión contextual, extracción financiera y análisis cualitativo

Financial Parser
= interpretación financiera mediante reglas deterministas
```

## Módulo 1 — Config & Logging

Centraliza la configuración y proporciona logging reutilizable. No
realiza interpretación financiera.

## Módulo 2 — Loaders

Carga PDF, DOCX, PPTX, TXT y texto pegado. Su responsabilidad es
obtener el contenido sin interpretar su significado financiero.

Limitación conocida: no realiza OCR.

## Módulo 3 — Text Extractor

Recibe la salida de los Loaders y produce una representación textual
uniforme. Incluye:

- normalización Unicode;
- normalización de espacios y saltos;
- unificación de separadores de página/slide;
- deduplicación acotada de ecos de tablas;
- detección heurística de headings;
- validación de contenido suficiente.

No busca vocabulario financiero ni crea candidatos.

## Módulo 4 — Document Analyzer

El Document Analyzer es exclusivamente estructural. Actualmente:

1. segmenta el texto en párrafos;
2. conserva sección, heading e indicador de tabla;
3. cierra correctamente los bloques en los cambios de sección;
4. evita heredar headings entre secciones;
5. convierte el heading en contexto lineal.

Ejemplo:

```text
# Revenue
It was 10% higher, to 100B.
```

se convierte en:

```text
Revenue: [It was 10% higher, to 100B.]
```

El Document Analyzer no:

- busca términos financieros;
- clasifica relevancia;
- crea candidatos;
- realiza scoring;
- interpreta números;
- decide si un movimiento es bueno o malo.

## Módulo 5 — Financial Candidate Finder

El Financial Candidate Finder utiliza directamente
`data/financial_terms/Financial_Terms_Synonyms.txt` como su fuente de
vocabulario.

Para cada coincidencia genera un candidato con:

```text
Candidate #1
Matched_metric: REVENUE
Evidence: "Revenue was 10% higher, to 100B."
Matched_term: "revenue"
```

También conserva metadata interna de trazabilidad:

- texto literal que coincidió;
- índice de párrafo;
- sección;
- heading;
- indicador de tabla.

El FCF no:

- interpreta economía;
- interpreta números;
- resuelve el significado financiero de una dirección;
- descarta subtipos;
- hace scoring;
- llama al LLM.

### Evidencia repetida

Dos candidatos distintos pueden compartir una misma evidencia:

```text
Revenue and net income increased 10% and 5%.
```

El FCF conserva ambos candidatos, pero crea un `EvidenceGroup` para que
la evidencia se entregue una sola vez al único LLM con los dos IDs de
candidato asociados.

### Segmentación de evidencia

El splitter sigue estas reglas:

- si después de un punto aparece un número, no corta la evidencia;
- un salto de línea siempre inicia una evidencia nueva;
- esto vale aunque la nueva línea comience con minúscula, número,
  asterisco o enumeración;
- una puntuación seguida por minúscula en la misma línea no corta;
- se protegen decimales y abreviaturas comunes.

Una misma evidencia puede producir varios candidatos.

## Único LLM

Es la próxima capa del pipeline. Recibirá grupos de evidencia y
candidatos asociados. Deberá producir Financial Fact Candidates sin
inventar información.

Por cada hecho podrá extraer:

- término;
- valor actual;
- valor anterior;
- variación porcentual;
- variación numérica;
- dirección lingüística normalizada;
- temporalidad;
- modificadores;
- incertidumbre;
- evidencia interna.

También será responsable del análisis cualitativo. No debe realizar el
scoring final ni reemplazar las reglas del Financial Parser.

## Financial Parser

Recibirá los Financial Fact Candidates estructurados. Será responsable
de:

- interpretar financieramente la dirección;
- determinar el signo financiero;
- calcular consecuencias económicas según reglas;
- preparar señales para las etapas posteriores.

No debe volver a resolver libremente problemas de segmentación o
comprensión lingüística que corresponden al único LLM.

## Regla de integridad

Nunca inventar información. Si un dato no está disponible o una
asociación no puede establecerse con seguridad, debe conservarse como
no disponible o incierta.

Es preferible una respuesta incompleta pero sustentada que una
respuesta completa pero incorrecta.

## Desarrollo

Cada etapa debe:

1. implementarse;
2. probarse;
3. pasar regresión;
4. documentarse;
5. verificarse;
6. entregarse con el proyecto completo actualizado en ZIP.

La README, `architecture_updated.md`, `roadmap_updated.md` y
`requirements_updated.txt` deben actualizarse junto con cada cambio de
arquitectura.
