# Content Creator — Multi-Agent Social Media Pipeline

**Agencia de contenido multi-agente que transforma textos densos en hilos virales para redes sociales.**  
**Multi-agent content agency that converts dense text into viral social media threads.**

---

## Table of Contents / Índice

- [Overview](#overview)
- [Architecture](#architecture)
- [Node Breakdown](#node-breakdown)
- [Design Decisions](#design-decisions)
- [Setup & Credentials](#setup--credentials)
- [Usage Example](#usage-example)
- [Limitations & Known Issues](#limitations--known-issues)
- [Didactic Notes](#didactic-notes)

---

## Overview

Este flujo implementa una **arquitectura Supervisor–Especialistas** donde un agente director (Claude Sonnet 4.5) coordina tres agentes ejecutores (GPT-3.5 Turbo) en una cadena secuencial:

```
Analista → Redactor → Editor
```

El resultado es un hilo de Twitter/X listo para publicar, con cada bloque respetando el límite de 280 caracteres.

This workflow implements a **Supervisor–Specialists architecture** where a director agent (Claude Sonnet 4.5) coordinates three executor agents (GPT-3.5 Turbo) in a strict sequential pipeline. Output: a publish-ready Twitter/X thread with each block under 280 characters.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     LangChain Sub-graph                          │
│                                                                  │
│  Chat Input                                                       │
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

> **Nota de diagrama / Diagram note:** Las conexiones `ai_tool` son parte del sub-grafo LangChain de n8n y no representan flujo secuencial estándar. El supervisor decide el orden de invocación en runtime.  
> `ai_tool` connections are part of n8n's LangChain sub-graph — they don't represent standard main-flow sequencing. The supervisor decides invocation order at runtime.

---

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
Recibe texto bruto → devuelve lista numerada de conceptos clave + identifica el gancho principal. Sin introducciones ni despedidas.

**`agente_redactor`**  
Recibe lista de puntos → escribe hilo narrativo en tono "edutainment". Reglas: oraciones cortas, espacio en blanco entre párrafos, máximo 1 emoji por punto, voz activa.

**`agente_editor`**  
Recibe borrador → corrige ortografía/gramática, asegura que ningún bloque supere 280 caracteres, elimina relleno. **Solo devuelve el texto corregido** — sin comentarios editoriales.

---

## Design Decisions

### Por qué Claude para el supervisor y GPT-3.5 para los especialistas

El supervisor requiere capacidad de planificación, razonamiento sobre el estado de la tarea y decisión de cuándo re-invocar a un especialista si el output no es satisfactorio. Claude Sonnet 4.5 tiene mejor rendimiento en este rol de meta-razonamiento.

Los especialistas ejecutan tareas bien definidas y atómicas donde GPT-3.5 Turbo es suficiente. Esto reduce el costo por ejecución — el supervisor es la única llamada a un modelo premium.

> The supervisor needs planning and state-reasoning capacity. Specialists execute well-scoped, atomic tasks where GPT-3.5 is sufficient. This reduces cost per run while maintaining quality where it matters.

### Por qué tools y no un chain lineal

Usar `agentTool` en lugar de un chain secuencial fijo le da al supervisor la capacidad de re-invocar a un agente si el output es pobre, o de saltarse un paso si el input ya viene estructurado. El flujo es más robusto ante inputs variables.

### Por qué un solo modelo compartido para los tres especialistas

Un único nodo `GPT 3.5 Turbo` conectado a los tres tools reduce el mantenimiento de credenciales y hace el flujo más legible. El comportamiento diferenciado viene del system prompt de cada tool, no del modelo.

---

## Setup & Credentials

| Credential | Node | Notes |
|-----------|------|-------|
| `anthropicApi` | Sonnet 4.5 | API key de Anthropic |
| `openAiApi` | GPT 3.5 Turbo | API key de OpenAI — usada por los 3 especialistas |

🟡 **El flujo no tiene memory node.** Cada ejecución es stateless — el supervisor no recuerda conversaciones anteriores. Adecuado para hilos one-shot; si necesitas iteración sobre el mismo tema, agrega un `Window Buffer Memory`.

---

## Usage Example

**Input:**
```
Convierte este artículo en un hilo viral: [pega texto o pega la URL del artículo]
```

**Expected output:**
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

## Limitations & Known Issues

🔴 **Sin validación del límite de 280 caracteres fuera del LLM.** El `agente_editor` aplica la restricción vía instrucción al modelo, pero no hay un nodo de validación determinista. Si el modelo produce un bloque largo, no hay fallback automático. Para producción, considera agregar un Code node de verificación post-editor.

🟡 **GPT-3.5 Turbo y coherencia narrativa.** En textos técnicos complejos, el `agente_redactor` puede simplificar en exceso. Si el dominio es técnico/especializado, evalúa subir a GPT-4o-mini.

🟡 **El supervisor no tiene memoria de sesión.** Múltiples turns de chat no acumulan contexto. El flujo está diseñado para ejecuciones one-shot.

---

## Didactic Notes

Este flujo ilustra tres patrones arquitectónicos relevantes para estudiantes de sistemas multi-agente:

1. **División estratégica / operativa.** El supervisor no ejecuta — planifica. Los especialistas no planifican — ejecutan. Esta separación reduce la carga cognitiva de cada agente y hace los fallos más rastreables.

2. **Costo vs. calidad por capa.** No todos los nodos requieren el modelo más potente. El costo se concentra donde el razonamiento es más complejo (supervisión), no donde la tarea es más repetible.

3. **System prompts como contrato.** Cada tool tiene un system prompt que define exactamente qué recibe, qué produce y en qué formato. Esto es la interfaz del agente — tan importante como la lógica del supervisor.