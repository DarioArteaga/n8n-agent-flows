# basic-crm

> **n8n workflow** — Lead capture pipeline with duplicate detection, AI scoring, and dual email notifications via Gmail.

---

**Language / Idioma:** [English](#english) · [Español](#español)

---

<a name="english"></a>
# 🇺🇸 English

## Table of Contents

1. [Overview](#overview-en)
2. [Architecture](#architecture-en)
3. [Node-by-Node Breakdown](#nodes-en)
4. [Design Decisions](#design-decisions-en)
5. [Prerequisites & Credentials](#prerequisites-en)
6. [Supabase Table Setup](#supabase-en)
7. [Limitations & Known Issues](#limitations-en)
8. [Didactic Notes](#didactic-en)

---

<a name="overview-en"></a>
## Overview

This workflow implements a **minimal viable CRM capture layer** built entirely in n8n. It handles the full lifecycle of an inbound lead: form submission → deduplication → AI scoring → database insert → dual email notification.

The pipeline is triggered by n8n's native form — no external form builder or webhook required.

```
[Form Trigger] → [Duplicate check — Supabase GET] → [AI Scoring — LLM Chain] → [Supabase INSERT] → [Email: agent + lead]
```

> **What this is not:** A full CRM. n8n is the orchestration layer — Supabase is the source of truth. This workflow captures and routes leads; it does not handle pipeline stages, deal tracking, or contact updates.

---

<a name="architecture-en"></a>
## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              basic-crm                                    │
│                                                                           │
│  ┌─────────────────┐    ┌──────────────────┐    ┌──────────────────────┐ │
│  │ Formulario      │───▶│ Buscar Lead      │───▶│ Check duplicado      │ │
│  │ de Lead         │    │ duplicado        │    │ (IF node)            │ │
│  │ (Form Trigger)  │    │ (Supabase GET)   │    └──────────┬───────────┘ │
│  └─────────────────┘    └──────────────────┘               │             │
│                                                    ┌────────┴──────┐     │
│                                                  true            false   │
│                                                    │               │     │
│                                                    ▼               ▼     │
│                                        ┌─────────────────┐  ┌─────────┐ │
│                                        │ Scoring AI      │  │  Stop   │ │
│                                        │ del Lead        │  │  (Lead  │ │
│                                        │ (LLM Chain)     │  │  dup.)  │ │
│                                        └────────┬────────┘  └─────────┘ │
│                                                 │                        │
│                              ┌──────────────────┘                        │
│                              │  LangChain Sub-graph                      │
│                              │  ┌─────────────────┐                      │
│                              │  │ Anthropic Chat  │ (ai_languageModel)   │
│                              │  │ Model           │                      │
│                              │  └─────────────────┘                      │
│                              │  ┌─────────────────┐                      │
│                              │  │ Parser Scoring  │ (ai_outputParser)    │
│                              │  └─────────────────┘                      │
│                              │                                            │
│                              ▼                                            │
│                   ┌──────────────────────┐                               │
│                   │ Insertar Lead        │                               │
│                   │ en Supabase          │                               │
│                   │ (Supabase INSERT)    │                               │
│                   └──────────┬───────────┘                               │
│                              │                                            │
│               ┌──────────────┴──────────────┐                            │
│               ▼                             ▼                            │
│  ┌────────────────────────┐  ┌──────────────────────────┐               │
│  │ Email al agente        │  │ Email de confirmación    │               │
│  │ de ventas (Gmail)      │  │ al Lead (Gmail)          │               │
│  └────────────────────────┘  └──────────────────────────┘               │
└──────────────────────────────────────────────────────────────────────────┘
```

> **Diagram note:** `Anthropic Chat Model` and `Parser Scoring` connect to the `Scoring AI del Lead` node via `ai_languageModel` and `ai_outputParser` connection types — part of n8n's LangChain sub-graph, not standard main-flow connections.

---

<a name="nodes-en"></a>
## Node-by-Node Breakdown

### 1. `Formulario de Lead`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.formTrigger` |
| Version | 2.5 |
| Purpose | Entry point — renders native n8n form and passes submission data downstream |
| Credentials | None |

Captures four fields: `nombre` (required), `email` (required, type `email`), `empresa` (optional), `mensaje` (optional). The `email` field type enforces format validation at the browser level.

> The code-level email validation node was intentionally omitted — see [Design Decisions](#design-decisions-en).

---

### 2. `Buscar Lead duplicado`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.supabase` |
| Operation | `row → get` |
| Table | `leads` |
| Filter | `email = $json.email` |
| `alwaysOutputData` | `true` |
| Credentials | `supabaseApi` |

Queries the `leads` table for an existing row matching the submitted email. `alwaysOutputData: true` is critical — without it, a no-match result returns an empty array and silently kills all downstream nodes.

---

### 3. `Check duplicado`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.if` |
| Version | 2.2 |
| Condition | `$json.id` is empty (string) |

Routes the execution based on whether a lead already exists. `true` branch (empty id = new lead) continues to scoring. `false` branch (id present = duplicate) routes to `Stop`.

---

### 4. `Stop (Lead duplicado)`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.stopAndError` |
| Error type | `errorMessage` |

Throws an explicit, visible error when a duplicate is detected. This is a deliberate choice over silent filtering — see [Design Decisions](#design-decisions-en).

---

### 5. `Scoring AI del Lead`
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.chainLlm` |
| Version | 1.9 |
| Prompt type | `define` |
| Output parser | Structured (JSON) |

Receives `nombre`, `empresa`, and `mensaje` from the form and asks the LLM to return a structured score. The prompt explicitly instructs the model to respond only with JSON — enforced by the downstream parser.

---

### 6. `Anthropic Chat Model` *(sub-node)*
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.lmChatAnthropic` |
| Model | `claude-sonnet-4-6` |
| Credentials | `anthropicApi` |

Language model for the scoring chain. Model-agnostic by design — can be swapped for any compatible LangChain LM node without modifying the chain logic.

---

### 7. `Parser Scoring` *(sub-node)*
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.outputParserStructured` |
| Version | 1.3 |
| Schema type | `fromJson` |
| Schema example | `{ "score": 7, "label": "warm", "reasoning": "..." }` |

Parses and validates the LLM output against a fixed JSON schema. Output is wrapped in an `output` key — downstream nodes reference it as `$json.output.score`, `$json.output.label`, `$json.output.reasoning`.

---

### 8. `Insertar Lead en Supabase`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.supabase` |
| Operation | `row → create` |
| Table | `leads` |
| Data mode | `defineBelow` |
| Credentials | `supabaseApi` |

Inserts the full lead record including AI scoring fields. References the form trigger node directly for contact fields — avoids expression chaining through intermediate nodes.

---

### 9. `Email al agente de ventas`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.gmail` |
| Version | 2.2 |
| Operation | `message → send` |
| Format | HTML |
| Credentials | `gmailOAuth2` |

Sends the agent an HTML email with all lead fields plus the AI score, label, and reasoning. The subject line dynamically includes the lead's name and score for quick triage from the inbox.

---

### 10. `Email de confirmación al Lead`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.gmail` |
| Version | 2.2 |
| Operation | `message → send` |
| Format | HTML |
| Credentials | `gmailOAuth2` |

Sends a confirmation email to the lead's submitted address. Stateless — no personalization beyond name. Fires in parallel with the agent email from the same Supabase INSERT output.

---

<a name="design-decisions-en"></a>
## Design Decisions

### Why the email validation Code node was removed

The `email` field type in n8n's Form Trigger enforces format validation at the browser level before submission. A separate Code node with regex validation is only necessary if the form's webhook endpoint is called directly — bypassing the UI. For a controlled, internal CRM form this risk is negligible, and the extra node adds complexity without meaningful protection. The decision is documented here rather than left as an oversight.

### Why `alwaysOutputData: true` on the Supabase GET

n8n's silent empty-output behavior is the most common source of invisible bugs in workflows that query data conditionally. When a Supabase GET returns no rows, the default behavior is to pass zero items downstream — which silently skips all subsequent nodes with no error in the execution log. `alwaysOutputData: true` forces the node to always emit one item (with null fields), preserving the execution chain and making the IF branch logic deterministic.

### Why `Stop and Error` for duplicates instead of silent filtering

Filtering duplicates silently (by just not passing items forward) produces the same invisible-bug problem: the execution log shows a successful run, but nothing was actually inserted or sent. Using `Stop and Error` makes duplicate detections explicit and visible in the execution log — essential for debugging and for understanding whether the system is behaving as expected in production.

### Why model-agnostic LLM scoring

The LLM Chain is connected to an Anthropic node, but the chain itself has no dependency on any specific provider. Swapping the model requires only replacing the `Anthropic Chat Model` sub-node — the prompt, parser, and downstream expressions remain unchanged. This separation makes the scoring layer portable across OpenAI, Anthropic, or any other LangChain-compatible model.

### Why both emails fire from the INSERT output (not in parallel branches)

Both Gmail nodes receive their input from the Supabase INSERT output. This means they only fire after the lead has been successfully persisted — not before. Sending notifications before confirming the write would risk notifying about a lead that was never stored.

---

<a name="prerequisites-en"></a>
## Prerequisites & Credentials

| Credential Name | Type | Used By | Notes |
|---|---|---|---|
| Supabase Personal | `supabaseApi` | `Buscar Lead duplicado`, `Insertar Lead en Supabase` | Project URL + service role key from Supabase dashboard |
| Anthropic Personal | `anthropicApi` | `Anthropic Chat Model` | API key with access to Claude Sonnet |
| Gmail Personal | `gmailOAuth2` | Both Gmail nodes | Gmail account authorized via OAuth2 |

> Before activating the workflow, update the `sendTo` field in `Email al agente de ventas` — currently set to `tuagente@empresa.com` — with the actual sales agent email address.

---

<a name="supabase-en"></a>
## Supabase Table Setup

Run the following SQL in your Supabase project before activating the workflow:

```sql
CREATE TABLE leads (
  id              UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  nombre          TEXT NOT NULL,
  email           TEXT NOT NULL UNIQUE,
  empresa         TEXT,
  mensaje         TEXT,
  score           INTEGER,
  label           TEXT CHECK (label IN ('hot', 'warm', 'cold')),
  ai_reasoning    TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

**Design notes:**
- `email UNIQUE` — enforced at the DB level. The Supabase GET check in n8n is the first layer; this constraint is the safety net.
- `label CHECK` — restricts values to `hot`, `warm`, `cold`. If the LLM returns an unexpected value, the INSERT will fail explicitly rather than storing garbage data.
- No `updated_at` column — this workflow only performs INSERTs. Add it when a lead update flow is implemented.

---

<a name="limitations-en"></a>
## Limitations & Known Issues

🔴 **No retry logic on email failure.** If either Gmail node fails (rate limit, auth error, invalid address), the error is thrown but the lead has already been inserted in Supabase. There is no rollback and no retry. For production, add an error branch with alerting or a queue mechanism.

🟡 **AI scoring is a single LLM call with no validation beyond the output parser.** The parser enforces JSON structure, but does not validate that `score` is between 1–10 or that `label` is one of the allowed values at the LLM output level. The Supabase `CHECK` constraint on `label` acts as a hard backstop.

🟡 **The agent email address is hardcoded.** `tuagente@empresa.com` in `Email al agente de ventas` must be manually updated before activation. There is no environment variable or credential abstraction for this value.

🟡 **No lead update flow.** This workflow only handles new lead capture. There is no mechanism to update a lead's status, reassign it, or track follow-up actions within the same pipeline.

🟡 **Form is public by default.** n8n's Form Trigger generates a public URL. Consider enabling IP allowlisting (`options.ipWhitelist`) or basic auth if the form should not be accessible to the open internet.

---

<a name="didactic-en"></a>
## Didactic Notes

This workflow illustrates three patterns relevant to students building automation systems:

1. **The silent empty-output problem.** n8n does not raise an error when a node passes zero items — it simply stops execution. Any workflow that includes a conditional data lookup (GET, search, filter) must explicitly handle the empty case, either with `alwaysOutputData: true` or a dedicated error branch. This is n8n's most common source of invisible bugs.

2. **Stop and Error as an observability tool.** Using `stopAndError` for expected failure states (duplicates, invalid inputs) is not just defensive programming — it's a logging strategy. It makes the execution log reflect what actually happened, rather than showing a false green.

3. **Output parser as a contract.** The `Structured Output Parser` defines what the LLM is allowed to return. Treating LLM output as unstructured text and parsing it downstream with regex or string manipulation is brittle. Defining the schema upfront — and letting the parser enforce it — makes the AI layer as predictable as any other node in the flow.

---

## File Structure

```
n8n-agent-flows/
└── crm/
    └── basic-crm/
        ├── basic-crm.json     # Exportable n8n workflow
        └── README.md          # This file
```

---

*Part of the [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) repository — a collection of production-grade n8n workflows for AI systems, RAG pipelines, and process automation.*

---
---

<a name="español"></a>
# 🇲🇽 Español

## Índice

1. [Overview](#overview-es)
2. [Arquitectura](#architecture-es)
3. [Descripción de nodos](#nodes-es)
4. [Decisiones de diseño](#design-decisions-es)
5. [Prerrequisitos y credenciales](#prerequisites-es)
6. [Configuración de la tabla en Supabase](#supabase-es)
7. [Limitaciones y problemas conocidos](#limitations-es)
8. [Notas didácticas](#didactic-es)

---

<a name="overview-es"></a>
## Overview

Este workflow implementa una **capa mínima viable de captura de CRM** construida completamente en n8n. Maneja el ciclo completo de un lead entrante: envío del formulario → deduplicación → scoring AI → insert en base de datos → notificación dual por email.

El pipeline se activa con el formulario nativo de n8n — no se requiere form builder externo ni webhook.

```
[Form Trigger] → [Check duplicado — Supabase GET] → [Scoring AI — LLM Chain] → [Supabase INSERT] → [Email: agente + lead]
```

> **Lo que esto no es:** Un CRM completo. n8n es la capa de orquestación — Supabase es la fuente de verdad. Este workflow captura y enruta leads; no gestiona etapas del pipeline, seguimiento de deals ni actualizaciones de contactos.

---

<a name="architecture-es"></a>
## Arquitectura

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              basic-crm                                    │
│                                                                           │
│  ┌─────────────────┐    ┌──────────────────┐    ┌──────────────────────┐ │
│  │ Formulario      │───▶│ Buscar Lead      │───▶│ Check duplicado      │ │
│  │ de Lead         │    │ duplicado        │    │ (nodo IF)            │ │
│  │ (Form Trigger)  │    │ (Supabase GET)   │    └──────────┬───────────┘ │
│  └─────────────────┘    └──────────────────┘               │             │
│                                                    ┌────────┴──────┐     │
│                                                  true            false   │
│                                                    │               │     │
│                                                    ▼               ▼     │
│                                        ┌─────────────────┐  ┌─────────┐ │
│                                        │ Scoring AI      │  │  Stop   │ │
│                                        │ del Lead        │  │  (Lead  │ │
│                                        │ (LLM Chain)     │  │  dup.)  │ │
│                                        └────────┬────────┘  └─────────┘ │
│                                                 │                        │
│                              ┌──────────────────┘                        │
│                              │  Sub-grafo LangChain                      │
│                              │  ┌─────────────────┐                      │
│                              │  │ Anthropic Chat  │ (ai_languageModel)   │
│                              │  │ Model           │                      │
│                              │  └─────────────────┘                      │
│                              │  ┌─────────────────┐                      │
│                              │  │ Parser Scoring  │ (ai_outputParser)    │
│                              │  └─────────────────┘                      │
│                              │                                            │
│                              ▼                                            │
│                   ┌──────────────────────┐                               │
│                   │ Insertar Lead        │                               │
│                   │ en Supabase          │                               │
│                   │ (Supabase INSERT)    │                               │
│                   └──────────┬───────────┘                               │
│                              │                                            │
│               ┌──────────────┴──────────────┐                            │
│               ▼                             ▼                            │
│  ┌────────────────────────┐  ┌──────────────────────────┐               │
│  │ Email al agente        │  │ Email de confirmación    │               │
│  │ de ventas (Gmail)      │  │ al Lead (Gmail)          │               │
│  └────────────────────────┘  └──────────────────────────┘               │
└──────────────────────────────────────────────────────────────────────────┘
```

> **Nota de diagrama:** `Anthropic Chat Model` y `Parser Scoring` se conectan al nodo `Scoring AI del Lead` mediante los tipos de conexión `ai_languageModel` y `ai_outputParser` — parte del sub-grafo LangChain de n8n, no conexiones estándar del flujo principal.

---

<a name="nodes-es"></a>
## Descripción de nodos

### 1. `Formulario de Lead`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.formTrigger` |
| Versión | 2.5 |
| Propósito | Punto de entrada — renderiza el formulario nativo de n8n y pasa los datos del envío aguas abajo |
| Credenciales | Ninguna |

Captura cuatro campos: `nombre` (requerido), `email` (requerido, tipo `email`), `empresa` (opcional), `mensaje` (opcional). El tipo de campo `email` aplica validación de formato a nivel del navegador.

> El nodo de validación de email por código fue omitido intencionalmente — ver [Decisiones de diseño](#design-decisions-es).

---

### 2. `Buscar Lead duplicado`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.supabase` |
| Operación | `row → get` |
| Tabla | `leads` |
| Filtro | `email = $json.email` |
| `alwaysOutputData` | `true` |
| Credenciales | `supabaseApi` |

Consulta la tabla `leads` buscando una fila existente con el email enviado. `alwaysOutputData: true` es crítico — sin él, un resultado sin coincidencia devuelve un array vacío y silenciosamente detiene todos los nodos downstream.

---

### 3. `Check duplicado`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.if` |
| Versión | 2.2 |
| Condición | `$json.id` está vacío (string) |

Enruta la ejecución según si el lead ya existe. La rama `true` (id vacío = lead nuevo) continúa al scoring. La rama `false` (id presente = duplicado) enruta al `Stop`.

---

### 4. `Stop (Lead duplicado)`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.stopAndError` |
| Tipo de error | `errorMessage` |

Lanza un error explícito y visible cuando se detecta un duplicado. Es una decisión deliberada frente al filtrado silencioso — ver [Decisiones de diseño](#design-decisions-es).

---

### 5. `Scoring AI del Lead`
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.chainLlm` |
| Versión | 1.9 |
| Tipo de prompt | `define` |
| Output parser | Estructurado (JSON) |

Recibe `nombre`, `empresa` y `mensaje` del formulario y solicita al LLM un score estructurado. El prompt instruye explícitamente al modelo a responder solo con JSON — aplicado por el parser downstream.

---

### 6. `Anthropic Chat Model` *(sub-nodo)*
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.lmChatAnthropic` |
| Modelo | `claude-sonnet-4-6` |
| Credenciales | `anthropicApi` |

Modelo de lenguaje para la cadena de scoring. Agnóstico al modelo por diseño — puede reemplazarse por cualquier nodo LM compatible con LangChain sin modificar la lógica de la cadena.

---

### 7. `Parser Scoring` *(sub-nodo)*
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.outputParserStructured` |
| Versión | 1.3 |
| Tipo de schema | `fromJson` |
| Schema de ejemplo | `{ "score": 7, "label": "warm", "reasoning": "..." }` |

Parsea y valida el output del LLM contra un schema JSON fijo. El output queda envuelto en una clave `output` — los nodos downstream lo referencian como `$json.output.score`, `$json.output.label`, `$json.output.reasoning`.

---

### 8. `Insertar Lead en Supabase`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.supabase` |
| Operación | `row → create` |
| Tabla | `leads` |
| Modo de datos | `defineBelow` |
| Credenciales | `supabaseApi` |

Inserta el registro completo del lead incluyendo los campos de scoring AI. Referencia directamente el nodo del form trigger para los campos de contacto — evita encadenar expresiones a través de nodos intermedios.

---

### 9. `Email al agente de ventas`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.gmail` |
| Versión | 2.2 |
| Operación | `message → send` |
| Formato | HTML |
| Credenciales | `gmailOAuth2` |

Envía al agente un email HTML con todos los campos del lead más el score AI, etiqueta y razonamiento. El asunto incluye dinámicamente el nombre del lead y el score para triaje rápido desde el inbox.

---

### 10. `Email de confirmación al Lead`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.gmail` |
| Versión | 2.2 |
| Operación | `message → send` |
| Formato | HTML |
| Credenciales | `gmailOAuth2` |

Envía un email de confirmación a la dirección enviada por el lead. Sin estado — sin personalización más allá del nombre. Se dispara en paralelo con el email del agente desde el mismo output del INSERT de Supabase.

---

<a name="design-decisions-es"></a>
## Decisiones de diseño

### Por qué se eliminó el nodo Code de validación de email

El tipo de campo `email` en el Form Trigger de n8n aplica validación de formato a nivel del navegador antes del envío. Un nodo Code con validación por regex solo es necesario si el endpoint webhook del formulario se llama directamente — saltándose la UI. Para un formulario CRM controlado e interno, este riesgo es negligible, y el nodo extra añade complejidad sin una protección real. La decisión está documentada aquí en lugar de dejarse como un olvido.

### Por qué `alwaysOutputData: true` en el Supabase GET

El comportamiento silencioso de output vacío de n8n es la fuente más común de bugs invisibles en workflows que consultan datos condicionalmente. Cuando un Supabase GET no encuentra filas, el comportamiento por defecto es pasar cero items aguas abajo — lo que silenciosamente omite todos los nodos subsecuentes sin ningún error en el execution log. `alwaysOutputData: true` fuerza al nodo a emitir siempre un item (con campos null), preservando la cadena de ejecución y haciendo la lógica del IF determinista.

### Por qué `Stop and Error` para duplicados en lugar de filtrado silencioso

Filtrar duplicados silenciosamente (simplemente no pasando items hacia adelante) produce el mismo problema de bug invisible: el execution log muestra una ejecución exitosa, pero nada fue insertado ni enviado. Usar `Stop and Error` hace las detecciones de duplicados explícitas y visibles en el execution log — esencial para debugging y para entender si el sistema se comporta como se espera en producción.

### Por qué scoring LLM agnóstico al modelo

La LLM Chain está conectada a un nodo de Anthropic, pero la cadena en sí no tiene dependencia de ningún proveedor específico. Cambiar el modelo requiere solo reemplazar el sub-nodo `Anthropic Chat Model` — el prompt, el parser y las expresiones downstream permanecen sin cambios. Esta separación hace la capa de scoring portable entre OpenAI, Anthropic o cualquier otro modelo compatible con LangChain.

### Por qué ambos emails se disparan desde el output del INSERT (no en ramas paralelas)

Ambos nodos Gmail reciben su input desde el output del INSERT de Supabase. Esto significa que solo se disparan después de que el lead ha sido persistido exitosamente — no antes. Enviar notificaciones antes de confirmar la escritura arriesgaría notificar sobre un lead que nunca fue almacenado.

---

<a name="prerequisites-es"></a>
## Prerrequisitos y credenciales

| Nombre de credencial | Tipo | Usado por | Notas |
|---|---|---|---|
| Supabase Personal | `supabaseApi` | `Buscar Lead duplicado`, `Insertar Lead en Supabase` | URL del proyecto + service role key desde el dashboard de Supabase |
| Anthropic Personal | `anthropicApi` | `Anthropic Chat Model` | API key con acceso a Claude Sonnet |
| Gmail Personal | `gmailOAuth2` | Ambos nodos Gmail | Cuenta Gmail autorizada vía OAuth2 |

> Antes de activar el workflow, actualiza el campo `sendTo` en `Email al agente de ventas` — actualmente configurado como `tuagente@empresa.com` — con la dirección real del agente de ventas.

---

<a name="supabase-es"></a>
## Configuración de la tabla en Supabase

Ejecuta el siguiente SQL en tu proyecto de Supabase antes de activar el workflow:

```sql
CREATE TABLE leads (
  id              UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  nombre          TEXT NOT NULL,
  email           TEXT NOT NULL UNIQUE,
  empresa         TEXT,
  mensaje         TEXT,
  score           INTEGER,
  label           TEXT CHECK (label IN ('hot', 'warm', 'cold')),
  ai_reasoning    TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

**Notas de diseño:**
- `email UNIQUE` — aplicado a nivel de BD. El check de Supabase GET en n8n es la primera capa; esta constraint es la red de seguridad.
- `label CHECK` — restringe los valores a `hot`, `warm`, `cold`. Si el LLM devuelve un valor inesperado, el INSERT fallará explícitamente en lugar de almacenar datos incorrectos.
- Sin columna `updated_at` — este workflow solo realiza INSERTs. Agrégala cuando se implemente un flujo de actualización de leads.

---

<a name="limitations-es"></a>
## Limitaciones y problemas conocidos

🔴 **Sin lógica de reintento ante fallo de email.** Si alguno de los nodos Gmail falla (rate limit, error de auth, dirección inválida), el error se lanza pero el lead ya fue insertado en Supabase. No hay rollback ni reintento. Para producción, agrega una rama de error con alertas o un mecanismo de cola.

🟡 **El scoring AI es una sola llamada al LLM sin validación más allá del output parser.** El parser aplica la estructura JSON, pero no valida que `score` esté entre 1–10 o que `label` sea uno de los valores permitidos a nivel del output del LLM. La constraint `CHECK` de Supabase en `label` actúa como tope duro.

🟡 **La dirección del agente está hardcodeada.** `tuagente@empresa.com` en `Email al agente de ventas` debe actualizarse manualmente antes de la activación. No hay variable de entorno ni abstracción de credencial para este valor.

🟡 **Sin flujo de actualización de leads.** Este workflow solo maneja la captura de nuevos leads. No hay mecanismo para actualizar el estado de un lead, reasignarlo o rastrear acciones de seguimiento dentro del mismo pipeline.

🟡 **El formulario es público por defecto.** El Form Trigger de n8n genera una URL pública. Considera habilitar allowlisting de IP (`options.ipWhitelist`) o autenticación básica si el formulario no debe ser accesible desde internet abierto.

---

<a name="didactic-es"></a>
## Notas didácticas

Este workflow ilustra tres patrones relevantes para estudiantes que construyen sistemas de automatización:

1. **El problema del output vacío silencioso.** n8n no lanza un error cuando un nodo pasa cero items — simplemente detiene la ejecución. Cualquier workflow que incluya una consulta condicional de datos (GET, búsqueda, filtro) debe manejar explícitamente el caso vacío, ya sea con `alwaysOutputData: true` o con una rama de error dedicada. Esta es la fuente más común de bugs invisibles en n8n.

2. **Stop and Error como herramienta de observabilidad.** Usar `stopAndError` para estados de fallo esperados (duplicados, inputs inválidos) no es solo programación defensiva — es una estrategia de logging. Hace que el execution log refleje lo que realmente ocurrió, en lugar de mostrar un falso verde.

3. **El output parser como contrato.** El `Structured Output Parser` define qué le está permitido devolver al LLM. Tratar el output del LLM como texto no estructurado y parsearlo downstream con regex o manipulación de strings es frágil. Definir el schema de antemano — y dejar que el parser lo aplique — hace la capa AI tan predecible como cualquier otro nodo del flujo.

---

## Estructura de archivos

```
n8n-agent-flows/
└── crm/
    └── basic-crm/
        ├── basic-crm.json     # Workflow exportable de n8n
        └── README.md          # Este archivo
```

---

*Parte del repositorio [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) — una colección de workflows de n8n para sistemas AI, pipelines RAG y automatización de procesos.*