# Content Creator — Multi-Agent Social Media Pipeline

> **n8n workflow** — Supervisor–Specialists architecture: a director agent coordinates three executor agents (Analyst → Writer → Editor) to convert dense text into publish-ready Twitter/X threads under 280 characters per block.

---

**Language / Idioma:** [English](#english) · [Español](#español)

---

<a name="english"></a>
# 🇺🇸 English

## Table of Contents

1. [Overview](#overview-en)
2. [Architecture](#architecture-en)
3. [Node Breakdown](#nodes-en)
4. [Design Decisions](#design-en)
5. [Setup & Credentials](#setup-en)
6. [Usage Example](#usage-en)
7. [Limitations & Known Issues](#limitations-en)
8. [Didactic Notes](#didactic-en)
9. [File Structure](#file-structure-en)

---

<a name="overview-en"></a>
## Overview

This workflow implements a **Supervisor–Specialists architecture** where a director agent (Claude Sonnet 4.5) coordinates three executor agents (GPT-3.5 Turbo) in a strict sequential pipeline:

```
Analyst → Writer → Editor
```

Output: a publish-ready Twitter/X thread with each block under 280 characters.

---

<a name="architecture-en"></a>
## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     LangChain Sub-graph                          │
│                                                                  │
│  Chat Input                                                      │
│      │                                                           │
│      ▼                                                           │
│  agente_supervisor  ◄──── Sonnet 4.5 (LLM)                      │
│      │ (ai_tool ×3)                                              │
│      ├──────────────► agente_analista  ◄── GPT-3.5 Turbo        │
│      ├──────────────► agente_redactor  ◄── GPT-3.5 Turbo        │
│      └──────────────► agente_editor    ◄── GPT-3.5 Turbo        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

> **Diagram note:** `ai_tool` connections are part of n8n's LangChain sub-graph — they don't represent standard main-flow sequencing. The supervisor decides invocation order at runtime.

---

<a name="nodes-en"></a>
## Node Breakdown

| Node | Type | Model | Role |
|------|------|-------|------|
| `When chat message received` | Chat Trigger | — | Entry point — receives raw text or topic from user |
| `agente_supervisor` | AI Agent | Claude Sonnet 4.5 | Director — orchestrates specialist sequence, does not write content |
| `agente_analista` | Agent Tool | GPT-3.5 Turbo | Curator — extracts key points and identifies the hook |
| `agente_redactor` | Agent Tool | GPT-3.5 Turbo | Writer — converts key points to narrative thread with emojis |
| `agente_editor` | Agent Tool | GPT-3.5 Turbo | QA — corrects grammar, enforces 280-char limit, removes filler |
| `Sonnet 4.5` | LM Chat Anthropic | claude-sonnet-4-5 | Language model for supervisor |
| `GPT 3.5 Turbo` | LM Chat OpenAI | gpt-3.5-turbo | Shared language model for all three specialist tools |

### Agent System Prompts Summary

**`agente_analista`**
Receives raw text → returns a numbered list of key concepts + identifies the main hook. No introductions or farewells.

**`agente_redactor`**
Receives list of points → writes a narrative thread in an "edutainment" tone. Rules: short sentences, whitespace between paragraphs, max 1 emoji per point, active voice.

**`agente_editor`**
Receives draft → corrects spelling/grammar, ensures no block exceeds 280 characters, removes filler. **Returns only the corrected text** — no editorial comments.

---

<a name="design-en"></a>
## Design Decisions

### Why Claude for the supervisor and GPT-3.5 for the specialists

The supervisor requires planning capacity, reasoning about task state, and deciding when to re-invoke a specialist if output is unsatisfactory. Claude Sonnet 4.5 performs better in this meta-reasoning role.

Specialists execute well-defined, atomic tasks where GPT-3.5 Turbo is sufficient. This reduces cost per run — the supervisor is the only call to a premium model.

### Why tools and not a linear chain

Using `agentTool` instead of a fixed sequential chain gives the supervisor the ability to re-invoke an agent if output is poor, or skip a step if input is already structured. The flow is more robust to variable inputs.

### Why a single shared model for the three specialists

A single `GPT 3.5 Turbo` node connected to all three tools reduces credential maintenance and makes the flow more readable. Differentiated behavior comes from each tool's system prompt, not the model.

---

<a name="setup-en"></a>
## Setup & Credentials

| Credential | Node | Notes |
|-----------|------|-------|
| `anthropicApi` | Sonnet 4.5 | Anthropic API key |
| `openAiApi` | GPT 3.5 Turbo | OpenAI API key — used by all 3 specialists |

🟡 **The flow has no memory node.** Each execution is stateless — the supervisor does not remember previous conversations. Suitable for one-shot threads; if you need iteration on the same topic, add a `Window Buffer Memory`.

---

<a name="usage-en"></a>
## Usage Example

**Input:**
```
Convert this article into a viral thread: [paste text or article URL]
```

**Expected output:**
```
🧵 Thread: [Topic]

1/ [Hook — the most surprising insight]

2/ [Necessary context]

3/ [Key point with data]

...

n/ [CTA or closing]
```

Each block ≤ 280 characters.

---

<a name="limitations-en"></a>
## Limitations & Known Issues

🔴 **No deterministic 280-character validation.** The `agente_editor` applies the restriction via model instruction, but there is no deterministic validation node. If the model produces a long block, there is no automatic fallback. For production, consider adding a post-editor Code node for verification.

🟡 **GPT-3.5 Turbo and narrative coherence.** On complex technical texts, `agente_redactor` may oversimplify. If the domain is technical/specialized, consider upgrading to GPT-4o-mini.

🟡 **The supervisor has no session memory.** Multiple chat turns do not accumulate context. The flow is designed for one-shot executions.

---

<a name="didactic-en"></a>
## Didactic Notes

This flow illustrates three architectural patterns relevant to students of multi-agent systems:

1. **Strategic / operational division.** The supervisor does not execute — it plans. Specialists do not plan — they execute. This separation reduces each agent's cognitive load and makes failures more traceable.

2. **Cost vs. quality by layer.** Not all nodes require the most powerful model. Cost concentrates where reasoning is most complex (supervision), not where the task is most repeatable.

3. **System prompts as contracts.** Each tool has a system prompt that defines exactly what it receives, what it produces, and in what format. This is the agent's interface — as important as the supervisor's logic.

---

<a name="file-structure-en"></a>
## File Structure

```
n8n-agent-flows/
└── content-creator/
    ├── content-creator.json     # Exportable n8n workflow
    └── README.md                # This file
```

---

*Part of the [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) repository — a collection of n8n workflows for AI systems, RAG pipelines, and process automation.*

---
---

<a name="español"></a>
# 🇲🇽 Español

## Tabla de contenidos

1. [Descripción general](#descripcion-es)
2. [Arquitectura](#arquitectura-es)
3. [Descripción de nodos](#nodos-es)
4. [Decisiones de diseño](#diseno-es)
5. [Setup y credenciales](#setup-es)
6. [Ejemplo de uso](#uso-es)
7. [Limitaciones y problemas conocidos](#limitaciones-es)
8. [Notas didácticas](#didacticas-es)
9. [Estructura de archivos](#estructura-es)

---

<a name="descripcion-es"></a>
## Descripción general

Este flujo implementa una **arquitectura Supervisor–Especialistas** donde un agente director (Claude Sonnet 4.5) coordina tres agentes ejecutores (GPT-3.5 Turbo) en una cadena secuencial:

```
Analista → Redactor → Editor
```

El resultado es un hilo de Twitter/X listo para publicar, con cada bloque respetando el límite de 280 caracteres.

---

<a name="arquitectura-es"></a>
## Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│                     Sub-grafo LangChain                          │
│                                                                  │
│  Chat Input                                                      │
│      │                                                           │
│      ▼                                                           │
│  agente_supervisor  ◄──── Sonnet 4.5 (LLM)                      │
│      │ (ai_tool ×3)                                              │
│      ├──────────────► agente_analista  ◄── GPT-3.5 Turbo        │
│      ├──────────────► agente_redactor  ◄── GPT-3.5 Turbo        │
│      └──────────────► agente_editor    ◄── GPT-3.5 Turbo        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

> **Nota de diagrama:** Las conexiones `ai_tool` son parte del sub-grafo LangChain de n8n y no representan flujo secuencial estándar. El supervisor decide el orden de invocación en runtime.

---

<a name="nodos-es"></a>
## Descripción de nodos

| Nodo | Tipo | Modelo | Rol |
|------|------|--------|-----|
| `When chat message received` | Chat Trigger | — | Punto de entrada — recibe texto bruto o tema del usuario |
| `agente_supervisor` | AI Agent | Claude Sonnet 4.5 | Director — orquesta la secuencia de especialistas, no escribe contenido |
| `agente_analista` | Agent Tool | GPT-3.5 Turbo | Curador — extrae puntos clave e identifica el gancho |
| `agente_redactor` | Agent Tool | GPT-3.5 Turbo | Redactor — convierte puntos clave en hilo narrativo con emojis |
| `agente_editor` | Agent Tool | GPT-3.5 Turbo | QA — corrige gramática, aplica límite de 280 chars, elimina relleno |
| `Sonnet 4.5` | LM Chat Anthropic | claude-sonnet-4-5 | Modelo de lenguaje para el supervisor |
| `GPT 3.5 Turbo` | LM Chat OpenAI | gpt-3.5-turbo | Modelo compartido para los tres tools especialistas |

### Resumen de system prompts

**`agente_analista`**
Recibe texto bruto → devuelve lista numerada de conceptos clave + identifica el gancho principal. Sin introducciones ni despedidas.

**`agente_redactor`**
Recibe lista de puntos → escribe hilo narrativo en tono "edutainment". Reglas: oraciones cortas, espacio en blanco entre párrafos, máximo 1 emoji por punto, voz activa.

**`agente_editor`**
Recibe borrador → corrige ortografía/gramática, asegura que ningún bloque supere 280 caracteres, elimina relleno. **Solo devuelve el texto corregido** — sin comentarios editoriales.

---

<a name="diseno-es"></a>
## Decisiones de diseño

### Por qué Claude para el supervisor y GPT-3.5 para los especialistas

El supervisor requiere capacidad de planificación, razonamiento sobre el estado de la tarea y decisión de cuándo re-invocar a un especialista si el output no es satisfactorio. Claude Sonnet 4.5 tiene mejor rendimiento en este rol de meta-razonamiento.

Los especialistas ejecutan tareas bien definidas y atómicas donde GPT-3.5 Turbo es suficiente. Esto reduce el costo por ejecución — el supervisor es la única llamada a un modelo premium.

### Por qué tools y no un chain lineal

Usar `agentTool` en lugar de un chain secuencial fijo le da al supervisor la capacidad de re-invocar a un agente si el output es pobre, o de saltarse un paso si el input ya viene estructurado. El flujo es más robusto ante inputs variables.

### Por qué un solo modelo compartido para los tres especialistas

Un único nodo `GPT 3.5 Turbo` conectado a los tres tools reduce el mantenimiento de credenciales y hace el flujo más legible. El comportamiento diferenciado viene del system prompt de cada tool, no del modelo.

---

<a name="setup-es"></a>
## Setup y credenciales

| Credencial | Nodo | Notas |
|-----------|------|-------|
| `anthropicApi` | Sonnet 4.5 | API key de Anthropic |
| `openAiApi` | GPT 3.5 Turbo | API key de OpenAI — usada por los 3 especialistas |

🟡 **El flujo no tiene memory node.** Cada ejecución es stateless — el supervisor no recuerda conversaciones anteriores. Adecuado para hilos one-shot; si necesitas iteración sobre el mismo tema, agrega un `Window Buffer Memory`.

---

<a name="uso-es"></a>
## Ejemplo de uso

**Input:**
```
Convierte este artículo en un hilo viral: [pega texto o pega la URL del artículo]
```

**Output esperado:**
```
🧵 Hilo: [Tema]

1/ [Gancho — el insight más sorprendente]

2/ [Contexto necesario]

3/ [Punto clave con dato]

...

n/ [CTA o cierre]
```

Cada bloque ≤ 280 caracteres.

---

<a name="limitaciones-es"></a>
## Limitaciones y problemas conocidos

🔴 **Sin validación del límite de 280 caracteres fuera del LLM.** El `agente_editor` aplica la restricción vía instrucción al modelo, pero no hay un nodo de validación determinista. Si el modelo produce un bloque largo, no hay fallback automático. Para producción, considera agregar un Code node de verificación post-editor.

🟡 **GPT-3.5 Turbo y coherencia narrativa.** En textos técnicos complejos, el `agente_redactor` puede simplificar en exceso. Si el dominio es técnico/especializado, evalúa subir a GPT-4o-mini.

🟡 **El supervisor no tiene memoria de sesión.** Múltiples turns de chat no acumulan contexto. El flujo está diseñado para ejecuciones one-shot.

---

<a name="didacticas-es"></a>
## Notas didácticas

Este flujo ilustra tres patrones arquitectónicos relevantes para estudiantes de sistemas multi-agente:

1. **División estratégica / operativa.** El supervisor no ejecuta — planifica. Los especialistas no planifican — ejecutan. Esta separación reduce la carga cognitiva de cada agente y hace los fallos más rastreables.

2. **Costo vs. calidad por capa.** No todos los nodos requieren el modelo más potente. El costo se concentra donde el razonamiento es más complejo (supervisión), no donde la tarea es más repetible.

3. **System prompts como contrato.** Cada tool tiene un system prompt que define exactamente qué recibe, qué produce y en qué formato. Esto es la interfaz del agente — tan importante como la lógica del supervisor.

---

<a name="estructura-es"></a>
## Estructura de archivos

```
n8n-agent-flows/
└── content-creator/
    ├── content-creator.json     # Workflow exportable de n8n
    └── README.md                # Este archivo
```

---

*Parte del repositorio [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) — una colección de workflows de n8n para sistemas AI, pipelines RAG y automatización de procesos.*
