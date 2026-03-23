# ingest_RAG

> **n8n workflow** — Manual RAG ingestion pipeline: downloads a document from Google Drive, chunks it, generates embeddings, and inserts vectors into a Supabase table ready for semantic search.

---

**Language / Idioma:** [English](#english) · [Español](#español)

---

<a name="english"></a>
# 🇺🇸 English

## Table of Contents

1. [Overview](#overview-en)
2. [Architecture](#architecture-en)
3. [Node-by-Node Breakdown](#nodes-en)
4. [Prerequisites & Credentials](#prerequisites-en)
5. [Technical Specs](#specs-en)
6. [Text Splitter: Design Decision & Trade-offs](#splitter-en)
7. [Limitations & Warnings](#limitations-en)
8. [Recommended Use Cases](#usecases-en)
9. [Extension Ideas](#extensions-en)

---

<a name="overview-en"></a>
## Overview

This workflow implements the **ingestion stage** of a Retrieval-Augmented Generation (RAG) system. Its sole responsibility is to populate a vector database with semantic chunks derived from a document — it does not handle querying, retrieval, or generation.

The pipeline is intentionally simple and **manually triggered**, making it suitable for one-time or infrequent knowledge base updates where full automation is not yet required.

```
[Manual Trigger] → [Download document from Drive] → [Chunk + Embed + Insert into Supabase]
```

> **Note on the source file:** The workflow comes configured with a placeholder Google Drive file ID. You will need to connect your own Google Drive account and select a supported file — see [Prerequisites](#prerequisites-en) for details.

---

<a name="architecture-en"></a>
## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          ingest_RAG                                  │
│                                                                       │
│  ┌──────────────┐    ┌───────────────┐    ┌────────────────────────┐│
│  │manual_trigger│───▶│ download_file │───▶│  populate_vectorial_db ││
│  └──────────────┘    └───────────────┘    └──────────┬─────────────┘│
│                        Google Drive                   │              │
│                        (binary file)         ┌────────┼────────┐    │
│                                              │        │        │    │
│                                    ┌─────────┴──┐ ┌───┴────┐   │    │
│                                    │embeddings_ │ │ data_  │   │    │
│                                    │3_small     │ │ loader │   │    │
│                                    │(OpenAI)    │ └───┬────┘   │    │
│                                    └────────────┘     │        │    │
│                                                ┌──────┴──────┐ │    │
│                                                │ recursive_  │ │    │
│                                                │ char_splitt.│ │    │
│                                                └─────────────┘ │    │
└─────────────────────────────────────────────────────────────────────┘
                              Vector Store: Supabase (pgvector)
                              Table: documentos_soporte
```

> The three sub-nodes (`embeddings_3_small`, `data_loader`, `recursive_character_text_splitter`) are **LangChain sub-graph nodes** that connect to the vector store via dedicated `ai_embedding`, `ai_document`, and `ai_textSplitter` connection types — not standard n8n main connections.

---

<a name="nodes-en"></a>
## Node-by-Node Breakdown

### 1. `manual_trigger`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.manualTrigger` |
| Purpose | Starts the workflow on-demand from the n8n UI |
| Credentials | None |

Execution is fully manual. There is no schedule, webhook, or event trigger. This is appropriate for controlled, infrequent knowledge base updates.

---

### 2. `download_file`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.googleDrive` |
| Operation | `download` |
| File | Configured via Google Drive file picker |
| Output format | Binary |
| Credentials | `googleDriveOAuth2Api` |

Downloads the target document as a binary file from Google Drive. **You must connect your own Google Drive account and select your own file** — the workflow does not include a shared or public document.

**Supported file types** (by the downstream `data_loader`):

| Format | Notes |
|---|---|
| `.pdf` | Must have a text layer (not scanned/image-only) |
| `.txt` | Plain text, best chunking results |
| `.docx` | Word documents |
| `.md` | Markdown files |
| `.csv` | Flat text extraction, no tabular awareness |

> ⚠️ Scanned PDFs (image-based, no embedded text) will produce empty or garbled output at the loader stage. Use a PDF with a proper text layer, or pre-process with OCR before ingestion.

---

### 3. `populate_vectorial_db`
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.vectorStoreSupabase` |
| Mode | `insert` |
| Target table | `documentos_soporte` |
| Credentials | `supabaseApi` |

Orchestrates the full LangChain ingestion chain: receives the raw binary document, coordinates chunking and embedding via its sub-nodes, and writes the resulting vectors to Supabase.

---

### 4. `embeddings_3_small` *(sub-node)*
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.embeddingsOpenAi` |
| Model | `text-embedding-3-small` |
| Output dimensions | 1536 |
| Credentials | `openAiApi` |

Generates dense vector representations for each text chunk. `text-embedding-3-small` offers a strong quality/cost balance — it outperforms `ada-002` on most benchmarks while costing less per token.

---

### 5. `data_loader` *(sub-node)*
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` |
| Data type | `binary` |

Parses the raw binary file into LangChain `Document` objects and passes them to the text splitter.

---

### 6. `recursive_character_text_splitter` *(sub-node)*
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter` |
| Chunk size | `1000` *(default)* |
| Chunk overlap | `200` |

Splits documents into overlapping text chunks. See the [dedicated section](#splitter-en) for a full analysis of this choice.

---

<a name="prerequisites-en"></a>
## Prerequisites & Credentials

### Required Credentials in n8n

| Credential Name | Type | Used By | How to Set Up |
|---|---|---|---|
| Google Drive (your account) | `googleDriveOAuth2Api` | `download_file` | [n8n Google Drive Docs](https://docs.n8n.io/integrations/builtin/credentials/google/) |
| Supabase | `supabaseApi` | `populate_vectorial_db` | Project URL + service role key from Supabase dashboard |
| OpenAI | `openAiApi` | `embeddings_3_small` | API key with embedding model access |

### Connecting Your Google Drive File

1. Open the `download_file` node in your n8n instance
2. Connect your Google Drive account via OAuth2
3. In the **File** field, use the file picker to select your document
4. Ensure the document is in a [supported format](#2-download_file)

### Supabase Table Setup

Before running the workflow, create the target table with pgvector support. Run the following in your Supabase SQL editor:

```sql
-- Enable pgvector extension (once per project)
CREATE EXTENSION IF NOT EXISTS vector;

-- Create the documents table
CREATE TABLE documentos_soporte (
  id        BIGSERIAL PRIMARY KEY,
  content   TEXT,
  metadata  JSONB,
  embedding VECTOR(1536)  -- matches text-embedding-3-small output dimensions
);

-- Index for efficient cosine similarity search
CREATE INDEX ON documentos_soporte
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

> If you rename the table, update the `populate_vectorial_db` node accordingly.

---

<a name="specs-en"></a>
## Technical Specs

| Component | Technology |
|---|---|
| Workflow engine | n8n (`executionOrder: v1`) |
| Vector store | Supabase with pgvector |
| Embedding model | OpenAI `text-embedding-3-small` (1536 dims) |
| Text splitter | Recursive Character — chunk: 1000 / overlap: 200 |
| Source | Google Drive (user-connected) |
| LangChain integration | `@n8n/n8n-nodes-langchain` |

---

<a name="splitter-en"></a>
## Text Splitter: Design Decision & Trade-offs

The splitter determines **what a "chunk" means** — and chunks are the atomic unit of retrieval. A poor chunking strategy is invisible at ingestion time but degrades retrieval quality significantly.

### The Three Options Available in n8n

#### 1. `Character Text Splitter`
Splits text based on a fixed character count using a single separator (default: `\n\n`).

| Pros | Cons |
|---|---|
| Simple and predictable | Rigid — splits regardless of sentence or paragraph boundaries |
| Zero overhead | Single separator: one misconfiguration breaks all splits |
| Works well for uniformly structured text | No fallback when separator is absent |

**Best for:** Structured documents with consistent delimiters (e.g., CSV exports, formatted logs).

---

#### 2. `Token Text Splitter`
Splits based on **token count** using a tokenizer (typically `tiktoken` / `cl100k_base`).

| Pros | Cons |
|---|---|
| Precise alignment with model token limits | Slower — requires tokenization at ingestion time |
| Prevents silent truncation during embedding | May split mid-sentence or mid-word |
| Accurate cost estimation per chunk | Token counts vary by model family and language |

**Best for:** Workflows where token budget control is critical, or high-volume batch embedding where model input limits must be respected exactly.

---

#### 3. `Recursive Character Text Splitter` *(used in this workflow)*
Attempts to split using a **priority-ordered list of separators**: `["\n\n", "\n", " ", ""]`. If chunks are still too large after the first separator, it falls back to the next one recursively.

```
Priority order: paragraph → line → word → character
```

| Pros | Cons |
|---|---|
| Respects natural text structure when possible | Still character-based — no semantic awareness |
| Graceful fallback prevents oversized chunks | Overlap is character-counted, not concept-counted |
| Better default for unstructured or mixed-format text | Can still split across logical boundaries |
| Widely adopted, well-tested for general RAG use | Character count ≠ token count |

**Best for:** General-purpose documents, PDFs with mixed structure, cases where document format is not fully controlled.

---

### Comparison Table

| Feature | Character Splitter | Token Splitter | **Recursive Character** |
|---|---|---|---|
| Split unit | Characters | Tokens | Characters (with fallback) |
| Structure awareness | ❌ None | ❌ None | ⚠️ Partial (whitespace) |
| Semantic awareness | ❌ | ❌ | ❌ |
| Token budget control | ❌ | ✅ Exact | ⚠️ Approximate |
| Handles PDFs well | ⚠️ Risky | ⚠️ Risky | ✅ Safer default |
| Complexity | Low | Medium | Low–Medium |

---

### ⚠️ Known Trade-offs of the Current Configuration

#### A. No semantic context awareness
Chunks are defined by character count, not meaning. A boundary at character 1000 can land in the middle of a multi-step procedure, a table row, or a concept defined across two paragraphs. The retriever can return a fragment that is syntactically present but semantically incomplete.

#### B. Overlap does not guarantee concept continuity
The 200-character overlap (~30–50 words) helps when a query spans a chunk boundary, but it is insufficient for multi-paragraph reasoning or dense technical content where concepts span entire sections.

#### C. Character count ≠ token count
With a 1000-character chunk and average English density (~4 chars/token), each chunk is approximately 250 tokens — well within `text-embedding-3-small`'s 8,191-token limit. However, chunks use only ~3% of the available context window. For denser documents or Spanish-language content, consider increasing `chunk_size` to 2000–3000 characters.

#### D. PDF text extraction quality dependency
The `data_loader` relies on the text layer of the PDF. Scanned documents, complex multi-column layouts, or embedded tables will produce noisy or disordered text before the splitter runs — and the splitter cannot compensate for that.

---

<a name="limitations-en"></a>
## Limitations & Warnings

### 🔴 No Deduplication
Running this workflow multiple times on the same document inserts **duplicate vectors** into Supabase. There is no hash check, no `upsert` mode, and no cleanup logic.

**Impact:** Retrieval returns duplicate chunks, bloating results and potentially skewing LLM context.

**Mitigation options:**
- Truncate the table before each run (`DELETE FROM documentos_soporte`)
- Add a `source_id` metadata field and delete by source before reinserting
- Add a Supabase query node to check for existing records before proceeding

---

### 🟡 Manual Trigger Only
Knowledge base updates require a human to initiate the workflow from the n8n UI.

**Acceptable when:** Documents change infrequently (e.g., quarterly manual updates).
**Not acceptable when:** The knowledge base needs near-real-time or event-driven updates.

**Mitigation:** Replace with a Google Drive `On File Updated` trigger.

---

### 🟡 Single-File, Single-Table Design
Processes exactly one file per run and writes to exactly one table. No support for batch ingestion or routing to different tables based on document type.

---

### 🟡 No Error Handling
If any node fails (auth error, connection timeout, rate limit), the workflow stops silently. There are no error branches, retry logic, or failure notifications.

---

### 🟡 Binary Mode: `separate`
Binary data is not stored inline with execution history. Correct for large files, but the source file is not preserved for post-run debugging.

---

<a name="usecases-en"></a>
## Recommended Use Cases

**Well-suited for:**
- ✅ Initial population of a support or documentation knowledge base
- ✅ One-off ingestion of a well-formatted document with a text layer
- ✅ RAG prototyping and proof-of-concept — fast to set up, easy to inspect
- ✅ Low-change-frequency documents: policies, manuals, product specs

**Not suited for:**
- ❌ High-frequency updates or real-time ingestion
- ❌ Multi-document or multi-source pipelines (without modification)
- ❌ Scanned PDFs or documents with complex visual layouts
- ❌ Production environments without deduplication and error handling

---

<a name="extensions-en"></a>
## Extension Ideas

| Enhancement | Approach |
|---|---|
| Multi-file ingestion | Loop over a Drive folder with a `SplitInBatches` node |
| Deduplication | Query Supabase for existing `source_id` before inserting |
| Automated trigger | Replace manual trigger with `Google Drive: On File Updated` |
| Semantic chunking | Add an LLM preprocessing step to split on concept boundaries |
| Token-aware chunking | Switch to `Token Text Splitter` with `chunk_size: 512` |
| Metadata enrichment | Inject filename, ingestion timestamp, and section ID into chunk metadata |
| Error notifications | Add error branch → Slack/email alert on failure |
| Multi-table routing | Add a `Switch` node to route to different tables by document type |

---
---

<a name="español"></a>
# 🇲🇽 Español

## Tabla de contenidos

1. [Descripción general](#descripcion-es)
2. [Arquitectura](#arquitectura-es)
3. [Descripción de nodos](#nodos-es)
4. [Requisitos y credenciales](#requisitos-es)
5. [Especificaciones técnicas](#specs-es)
6. [Text Splitter: decisión de diseño y trade-offs](#splitter-es)
7. [Limitaciones y advertencias](#limitaciones-es)
8. [Casos de uso recomendados](#casos-es)
9. [Ideas de extensión](#extensiones-es)

---

<a name="descripcion-es"></a>
## Descripción general

Este workflow implementa la **etapa de ingesta** de un sistema RAG (Retrieval-Augmented Generation). Su única responsabilidad es poblar una base de datos vectorial con fragmentos semánticos derivados de un documento — no maneja consultas, recuperación ni generación.

El pipeline es intencionalmente simple y **se activa de forma manual**, lo que lo hace adecuado para actualizaciones puntuales o poco frecuentes de la base de conocimientos donde la automatización completa aún no es necesaria.

```
[Trigger manual] → [Descarga documento de Drive] → [Fragmentar + Embeber + Insertar en Supabase]
```

> **Nota sobre el archivo fuente:** El workflow viene configurado con un ID de archivo de Google Drive de ejemplo. Deberás conectar tu propia cuenta de Google Drive y seleccionar tu propio archivo — consulta [Requisitos](#requisitos-es) para más detalles.

---

<a name="arquitectura-es"></a>
## Arquitectura

```
┌─────────────────────────────────────────────────────────────────────┐
│                          ingest_RAG                                  │
│                                                                       │
│  ┌──────────────┐    ┌───────────────┐    ┌────────────────────────┐│
│  │manual_trigger│───▶│ download_file │───▶│  populate_vectorial_db ││
│  └──────────────┘    └───────────────┘    └──────────┬─────────────┘│
│                        Google Drive                   │              │
│                        (archivo binario)     ┌────────┼────────┐    │
│                                              │        │        │    │
│                                    ┌─────────┴──┐ ┌───┴────┐   │    │
│                                    │embeddings_ │ │ data_  │   │    │
│                                    │3_small     │ │ loader │   │    │
│                                    │(OpenAI)    │ └───┬────┘   │    │
│                                    └────────────┘     │        │    │
│                                                ┌──────┴──────┐ │    │
│                                                │ recursive_  │ │    │
│                                                │ char_splitt.│ │    │
│                                                └─────────────┘ │    │
└─────────────────────────────────────────────────────────────────────┘
                              Base vectorial: Supabase (pgvector)
                              Tabla: documentos_soporte
```

> Los tres sub-nodos (`embeddings_3_small`, `data_loader`, `recursive_character_text_splitter`) son **nodos del sub-grafo LangChain** que se conectan al vector store mediante tipos de conexión dedicados: `ai_embedding`, `ai_document` y `ai_textSplitter` — no son conexiones `main` estándar de n8n.

---

<a name="nodos-es"></a>
## Descripción de nodos

### 1. `manual_trigger`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.manualTrigger` |
| Propósito | Inicia el workflow manualmente desde la UI de n8n |
| Credenciales | Ninguna |

La ejecución es completamente manual. No hay schedule, webhook ni trigger por evento. Es adecuado para actualizaciones controladas y poco frecuentes de la base de conocimientos.

---

### 2. `download_file`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.googleDrive` |
| Operación | `download` |
| Archivo | Se selecciona mediante el file picker de Google Drive |
| Formato de salida | Binario |
| Credenciales | `googleDriveOAuth2Api` |

Descarga el documento objetivo como archivo binario desde Google Drive. **Es necesario conectar tu propia cuenta de Google Drive y seleccionar tu propio archivo** — el workflow no incluye ningún archivo compartido o público.

**Formatos de archivo soportados** (por el `data_loader` aguas abajo):

| Formato | Notas |
|---|---|
| `.pdf` | Debe tener capa de texto (no escaneado ni solo imagen) |
| `.txt` | Texto plano, mejores resultados de fragmentación |
| `.docx` | Documentos Word |
| `.md` | Archivos Markdown |
| `.csv` | Extracción de texto plano, sin conciencia tabular |

> ⚠️ Los PDFs escaneados (basados en imagen, sin texto embebido) producirán salida vacía o con basura en la etapa del loader. Usa un PDF con capa de texto correcta, o aplica OCR antes de la ingesta.

---

### 3. `populate_vectorial_db`
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.vectorStoreSupabase` |
| Modo | `insert` |
| Tabla destino | `documentos_soporte` |
| Credenciales | `supabaseApi` |

Orquesta la cadena de ingesta completa de LangChain: recibe el documento binario, coordina el fragmentado y embebido mediante sus sub-nodos, y escribe los vectores resultantes en Supabase.

---

### 4. `embeddings_3_small` *(sub-nodo)*
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.embeddingsOpenAi` |
| Modelo | `text-embedding-3-small` |
| Dimensiones de salida | 1536 |
| Credenciales | `openAiApi` |

Genera representaciones vectoriales densas para cada fragmento de texto. `text-embedding-3-small` ofrece un balance sólido entre calidad y costo — supera a `ada-002` en la mayoría de benchmarks siendo más barato por token.

---

### 5. `data_loader` *(sub-nodo)*
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` |
| Tipo de dato | `binary` |

Parsea el archivo binario en objetos `Document` de LangChain y los pasa al text splitter.

---

### 6. `recursive_character_text_splitter` *(sub-nodo)*
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter` |
| Tamaño de chunk | `1000` *(por defecto)* |
| Overlap de chunk | `200` |

Divide los documentos en fragmentos de texto con solapamiento. Consulta la [sección dedicada](#splitter-es) para un análisis completo de esta elección.

---

<a name="requisitos-es"></a>
## Requisitos y credenciales

### Credenciales requeridas en n8n

| Nombre de credencial | Tipo | Usada por | Cómo configurar |
|---|---|---|---|
| Google Drive (tu cuenta) | `googleDriveOAuth2Api` | `download_file` | [Docs Google Drive n8n](https://docs.n8n.io/integrations/builtin/credentials/google/) |
| Supabase | `supabaseApi` | `populate_vectorial_db` | URL del proyecto + service role key desde el dashboard de Supabase |
| OpenAI | `openAiApi` | `embeddings_3_small` | API key con acceso al modelo de embeddings |

### Cómo conectar tu archivo de Google Drive

1. Abre el nodo `download_file` en tu instancia de n8n
2. Conecta tu cuenta de Google Drive mediante OAuth2
3. En el campo **File**, usa el file picker para seleccionar tu documento
4. Asegúrate de que el documento esté en un [formato soportado](#2-download_file-1)

### Configuración de la tabla en Supabase

Antes de ejecutar el workflow, crea la tabla destino con soporte de pgvector. Ejecuta lo siguiente en el editor SQL de Supabase:

```sql
-- Habilitar extensión pgvector (una sola vez por proyecto)
CREATE EXTENSION IF NOT EXISTS vector;

-- Crear la tabla de documentos
CREATE TABLE documentos_soporte (
  id        BIGSERIAL PRIMARY KEY,
  content   TEXT,
  metadata  JSONB,
  embedding VECTOR(1536)  -- coincide con las dimensiones de text-embedding-3-small
);

-- Índice para búsqueda eficiente por similitud coseno
CREATE INDEX ON documentos_soporte
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

> Si renombras la tabla, actualiza el nodo `populate_vectorial_db` en consecuencia.

---

<a name="specs-es"></a>
## Especificaciones técnicas

| Componente | Tecnología |
|---|---|
| Motor de workflow | n8n (`executionOrder: v1`) |
| Base de datos vectorial | Supabase con pgvector |
| Modelo de embeddings | OpenAI `text-embedding-3-small` (1536 dims) |
| Text splitter | Recursive Character — chunk: 1000 / overlap: 200 |
| Fuente | Google Drive (conectado por el usuario) |
| Integración LangChain | `@n8n/n8n-nodes-langchain` |

---

<a name="splitter-es"></a>
## Text Splitter: decisión de diseño y trade-offs

El splitter determina **qué significa un "chunk"** — y los chunks son la unidad atómica de recuperación. Una estrategia de fragmentación deficiente es invisible en el momento de la ingesta, pero degrada significativamente la calidad del retrieval.

### Las tres opciones disponibles en n8n

#### 1. `Character Text Splitter`
Divide el texto en función de un número fijo de caracteres usando un único separador (por defecto: `\n\n`).

| Ventajas | Desventajas |
|---|---|
| Simple y predecible | Rígido — divide sin importar límites de oraciones o párrafos |
| Sin overhead | Un solo separador: un error de configuración rompe todas las divisiones |
| Funciona bien para texto con estructura uniforme | Sin fallback cuando el separador no está presente |

**Ideal para:** Documentos estructurados con delimitadores consistentes (ej. exports CSV, logs formateados).

---

#### 2. `Token Text Splitter`
Divide en función del **conteo de tokens** usando un tokenizador (típicamente `tiktoken` / `cl100k_base`).

| Ventajas | Desventajas |
|---|---|
| Alineación precisa con los límites de tokens del modelo | Más lento — requiere tokenización en tiempo de ingesta |
| Previene truncamiento silencioso al embeber | Puede dividir a mitad de oración o de palabra |
| Estimación precisa del costo por chunk | El conteo de tokens varía según familia de modelo e idioma |

**Ideal para:** Workflows donde el control preciso del presupuesto de tokens es crítico, o ingestas en lote de alto volumen donde los límites de entrada del modelo deben respetarse exactamente.

---

#### 3. `Recursive Character Text Splitter` *(utilizado en este workflow)*
Intenta dividir usando una **lista de separadores en orden de prioridad**: `["\n\n", "\n", " ", ""]`. Si los chunks siguen siendo demasiado grandes tras aplicar el primer separador, cae al siguiente de forma recursiva.

```
Orden de prioridad: párrafo → línea → palabra → carácter
```

| Ventajas | Desventajas |
|---|---|
| Respeta la estructura natural del texto cuando es posible | Sigue siendo basado en caracteres — sin conciencia semántica |
| El fallback graceful evita chunks excesivamente grandes | El overlap se cuenta en caracteres, no en conceptos |
| Mejor opción por defecto para texto no estructurado o de formato mixto | Puede seguir dividiendo a lo largo de límites lógicos |
| Ampliamente adoptado, bien probado para RAG de propósito general | Conteo de caracteres ≠ conteo de tokens |

**Ideal para:** Documentos de propósito general, PDFs con estructura mixta, casos donde el formato del documento no está totalmente controlado.

---

### Tabla comparativa

| Característica | Character Splitter | Token Splitter | **Recursive Character** |
|---|---|---|---|
| Unidad de división | Caracteres | Tokens | Caracteres (con fallback) |
| Conciencia de estructura | ❌ Ninguna | ❌ Ninguna | ⚠️ Parcial (espacios en blanco) |
| Conciencia semántica | ❌ | ❌ | ❌ |
| Control de presupuesto de tokens | ❌ | ✅ Exacto | ⚠️ Aproximado |
| Manejo de PDFs | ⚠️ Arriesgado | ⚠️ Arriesgado | ✅ Opción más segura |
| Complejidad | Baja | Media | Baja–Media |

---

### ⚠️ Trade-offs conocidos de la configuración actual

#### A. Sin conciencia de contexto semántico
Los chunks se definen por conteo de caracteres, no por significado. Un límite en el carácter 1000 puede caer en medio de un procedimiento de múltiples pasos, una fila de tabla o un concepto definido a través de dos párrafos. El retriever puede devolver un fragmento que está sintácticamente presente pero semánticamente incompleto.

#### B. El overlap no garantiza continuidad conceptual
El overlap de 200 caracteres (~30–50 palabras) ayuda cuando una consulta abarca el límite de un chunk, pero es insuficiente para razonamiento de múltiples párrafos o contenido técnico denso donde los conceptos se extienden por secciones enteras.

#### C. Conteo de caracteres ≠ conteo de tokens
Con un chunk de 1000 caracteres y una densidad promedio en inglés (~4 chars/token), cada chunk tiene aproximadamente 250 tokens — muy dentro del límite de 8,191 tokens de `text-embedding-3-small`. Sin embargo, los chunks usan solo ~3% de la ventana de contexto disponible. Para documentos más densos o en español, considera aumentar `chunk_size` a 2000–3000 caracteres.

#### D. Dependencia de la calidad de extracción de texto del PDF
El `data_loader` depende de la capa de texto del PDF. Documentos escaneados, layouts de múltiples columnas o tablas embebidas producirán texto ruidoso o desordenado antes de que el splitter se ejecute — y el splitter no puede compensar eso.

---

<a name="limitaciones-es"></a>
## Limitaciones y advertencias

### 🔴 Sin deduplicación
Ejecutar este workflow varias veces sobre el mismo documento insertará **vectores duplicados** en Supabase. No hay verificación de hash, modo `upsert` ni lógica de limpieza.

**Impacto:** El retrieval devuelve chunks duplicados, inflando los resultados y potencialmente sesgando el contexto del LLM.

**Opciones de mitigación:**
- Truncar la tabla antes de cada ejecución (`DELETE FROM documentos_soporte`)
- Agregar un campo de metadatos `source_id` y eliminar por fuente antes de reinsertar
- Agregar un nodo de consulta a Supabase para verificar registros existentes antes de continuar

---

### 🟡 Solo trigger manual
Las actualizaciones de la base de conocimientos requieren que un humano inicie el workflow desde la UI de n8n.

**Aceptable cuando:** Los documentos cambian con poca frecuencia (ej. actualizaciones trimestrales de manuales).
**No aceptable cuando:** La base de conocimientos necesita actualizaciones en tiempo casi real o basadas en eventos.

**Mitigación:** Reemplazar con un trigger `On File Updated` de Google Drive.

---

### 🟡 Diseño de archivo único y tabla única
Procesa exactamente un archivo por ejecución y escribe en exactamente una tabla. No hay soporte para ingesta en lote ni enrutamiento a diferentes tablas según el tipo de documento.

---

### 🟡 Sin manejo de errores
Si algún nodo falla (error de autenticación, timeout de conexión, rate limit), el workflow se detiene silenciosamente. No hay ramas de error, lógica de reintentos ni notificaciones de falla.

---

### 🟡 Binary mode: `separate`
Los datos binarios no se almacenan en línea con el historial de ejecución. Correcto para archivos grandes, pero el archivo fuente no se conserva para depuración post-ejecución.

---

<a name="casos-es"></a>
## Casos de uso recomendados

**Adecuado para:**
- ✅ Población inicial de una base de conocimientos de soporte o documentación
- ✅ Ingesta puntual de un documento bien formateado con capa de texto
- ✅ Prototipado RAG y prueba de concepto — rápido de configurar, fácil de inspeccionar
- ✅ Documentos de baja frecuencia de cambio: políticas, manuales, especificaciones de productos

**No adecuado para:**
- ❌ Actualizaciones de alta frecuencia o ingesta en tiempo real
- ❌ Pipelines multi-documento o multi-fuente (sin modificación)
- ❌ PDFs escaneados o documentos con layouts visuales complejos
- ❌ Entornos de producción sin deduplicación y manejo de errores

---

<a name="extensiones-es"></a>
## Ideas de extensión

| Mejora | Enfoque |
|---|---|
| Ingesta multi-archivo | Iterar sobre una carpeta de Drive con un nodo `SplitInBatches` |
| Deduplicación | Consultar Supabase por `source_id` existente antes de insertar |
| Trigger automatizado | Reemplazar trigger manual con `Google Drive: On File Updated` |
| Fragmentación semántica | Agregar un paso de preprocesamiento con LLM para dividir por límites conceptuales |
| Fragmentación consciente de tokens | Cambiar a `Token Text Splitter` con `chunk_size: 512` |
| Enriquecimiento de metadatos | Inyectar nombre del archivo, timestamp de ingesta e ID de sección en los metadatos del chunk |
| Notificaciones de error | Agregar rama de error → alerta por Slack/email en caso de falla |
| Enrutamiento multi-tabla | Agregar nodo `Switch` para enrutar a diferentes tablas según el tipo de documento |

---

## Estructura de archivos

```
n8n-agent-flows/
└── rag/
    └── ingest_RAG/
        ├── ingest_RAG.json     # Exportable n8n workflow / Workflow exportable de n8n
        └── README.md           # This file / Este archivo
```

---

*Part of the [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) repository — a collection of production-grade n8n workflows for AI systems, RAG pipelines, and process automation.*

*Parte del repositorio [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) — una colección de workflows de n8n para sistemas AI, pipelines RAG y automatización de procesos.*
