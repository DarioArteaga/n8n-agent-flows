# agente-calculadora

> **n8n workflow** — Conversational agent with tool use: receives a user message through n8n's built-in chat UI, reasons about whether arithmetic is needed, delegates the calculation to a Calculator tool, and returns a response in Spanish. Designed as a classroom introduction to the agent tool-use loop.

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
6. [Design Decisions](#design-en)
7. [Limitations & Warnings](#limitations-en)
8. [Recommended Use Cases](#usecases-en)
9. [Extension Ideas](#extensions-en)

---

<a name="overview-en"></a>
## Overview

This workflow exposes a minimal AI agent through n8n's hosted chat interface. The agent has access to one tool — a Calculator — and must decide on each turn whether to invoke it or answer directly from its language model.

The pipeline is intentionally simple:

```
[Chat Trigger] → [AI Agent] ← [OpenAI Chat Model]
                            ← [Calculator Tool]
```

> **Note on credentials:** The workflow ships with a `newCredential('OpenAI API')` placeholder. You must connect your own OpenAI credential before the workflow will run — see [Prerequisites](#prerequisites-en) for details.

---

<a name="architecture-en"></a>
## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         agente-calculadora                                │
│                                                                            │
│  ┌─────────────────┐                                                      │
│  │ Chat Trigger    │                                                      │
│  │ (hosted UI)     │                                                      │
│  └────────┬────────┘                                                      │
│           │  chatInput                                                    │
│           ▼                                                               │
│  ┌────────────────────────────────────────────────────┐                  │
│  │                   AI Agent (ReAct)                  │                  │
│  │                                                     │                  │
│  │  ┌──────────────────────┐  ┌──────────────────┐   │                  │
│  │  │  OpenAI Chat Model   │  │  Calculator Tool  │   │                  │
│  │  │  (gpt-4o-mini)       │  │  (no credentials) │   │                  │
│  │  └──────────────────────┘  └──────────────────┘   │                  │
│  └────────────────────────────────────────────────────┘                  │
│           │  output                                                       │
│           ▼                                                               │
│       [Chat Response]                                                     │
└──────────────────────────────────────────────────────────────────────────┘
```

---

<a name="nodes-en"></a>
## Node-by-Node Breakdown

### 1. `Chat`
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.chatTrigger` v1.4 |
| Mode | Hosted Chat (n8n-served UI) |
| Session handling | Auto — each browser tab is an isolated session |
| Output | `chatInput` string passed to the agent |

Exposes n8n's built-in chat page and fires the workflow on each message submission. No external infrastructure required — the UI is served directly by the n8n instance.

> **Note:** The hosted chat URL is tied to the active workflow. The workflow must be activated (not just saved) for the chat endpoint to be reachable.

---

### 2. `Agente Asistente`
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.agent` v3.1 |
| Prompt source | `auto` — reads `chatInput` from the Chat Trigger |
| System message | See [Design Decisions](#design-en) |
| Max iterations | 5 |
| Intermediate steps | Not exposed (default) |
| Subnodes | `OpenAI Chat Model`, `Calculator` |

Orchestrates the ReAct (Reason + Act) loop. On each turn, the model decides whether to call the Calculator tool or respond directly. The iteration cap (5) prevents runaway loops on ambiguous inputs.

---

### 3. `OpenAI Chat Model`
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.lmChatOpenAi` v1.3 |
| Model | `gpt-4o-mini` (default) |
| Credentials | `openAiApi` (user-configured) |

Provides the language model backbone for the agent. `gpt-4o-mini` is selected for cost efficiency in a classroom context. Can be swapped for any model available through the OpenAI API.

---

### 4. `Calculator`
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.toolCalculator` v1 |
| Credentials | None required |
| Scope | Arithmetic expressions only |

Evaluates mathematical expressions passed by the agent. Runs entirely inside n8n — no external API call, no quota. The agent formulates an expression (e.g., `2400 * 0.15`) and the tool returns the numeric result.

> **Note:** The Calculator evaluates expressions, not natural language. The agent is responsible for translating the user's intent into a valid arithmetic expression before invoking the tool.

---

<a name="prerequisites-en"></a>
## Prerequisites & Credentials

### Required Credentials in n8n

| Credential Name | Type | Used By | How to Set Up |
|---|---|---|---|
| OpenAI API | `openAiApi` | `OpenAI Chat Model` | [n8n OpenAI Docs](https://docs.n8n.io/integrations/builtin/credentials/openai/) |

### Setup Steps

1. Import the workflow into your n8n instance.
2. Open the **OpenAI Chat Model** node and connect your OpenAI credential.
3. (Optional) Edit the system message in **Agente Asistente → Options → System Message**.
4. Activate the workflow.
5. Click the **Chat** button in the n8n toolbar to open the hosted UI.

---

<a name="specs-en"></a>
## Technical Specs

| Component | Technology |
|---|---|
| Workflow engine | n8n |
| Trigger | Chat Trigger (hosted UI) |
| Agent type | ReAct (Reason + Act) |
| Language model | OpenAI `gpt-4o-mini` |
| Tools | Calculator (n8n built-in) |
| Memory | None — stateless per message |
| Credentials required | OpenAI API |

---

<a name="design-en"></a>
## Design Decisions

### Why no memory?
This workflow is deliberately stateless. Adding memory introduces a second concept (session persistence) that competes for attention with the primary goal: making the tool-use loop visible. Memory is covered in the companion workflow `agente-investigador`.

### Why the Calculator and not a Code tool?
The Calculator tool has a fixed, well-understood scope — arithmetic — which makes the agent's decision boundary easy to reason about. A Code tool can execute arbitrary logic, which blurs the conceptual clarity needed in a first-session example.

### Why `promptType: auto`?
With `auto`, the agent reads `chatInput` directly from the Chat Trigger's output without requiring an explicit expression. This keeps the node configuration minimal and reduces the surface area for configuration errors during a live session.

### System message rationale
```
Eres un asistente util y amigable. Tienes acceso a una calculadora
para resolver operaciones matematicas. Responde siempre en espanol.
```
The prompt explicitly names the available tool. This is important: without an explicit mention, smaller models may underutilize the tool on problems they can partially solve in their heads — producing less accurate results while appearing to succeed.

### Why `maxIterations: 5`?
The default (10) is generous for a simple calculator agent. Five iterations is sufficient for any arithmetic problem reachable through this tool while capping compute cost and preventing loops on pathological inputs like circular reasoning chains.

---

<a name="limitations-en"></a>
## Limitations & Warnings

### 🔴 No Session Memory
Each message is processed independently. The agent has no awareness of previous turns in the same conversation.

**Impact:** The agent cannot answer follow-up questions that reference prior context. A user asking "what was that result again?" will receive a response based on the current message alone.

**Mitigation:** Add a `memoryBufferWindow` subnode to the agent and set `loadPreviousSession: memory` on the Chat Trigger. This is the architecture used in `agente-investigador`.

---

### 🟡 Model May Skip the Tool on Simple Expressions
`gpt-4o-mini` occasionally evaluates trivial arithmetic (e.g., `2 + 2`) without invoking the Calculator. This is not a failure — the model is capable of simple mental arithmetic — but it reduces the clarity of the demo.

**Impact:** Students may not see the tool-use loop on every query.

**Mitigation:** Use inputs that are clearly non-trivial, such as percentages, multi-step expressions, or square roots. See [recommended test inputs](#usecases-en).

---

### 🟡 Calculator Has No Unit or Context Awareness
The tool evaluates expressions as pure numbers. `15% of 2400 pesos` and `15% of 2400 km` produce the same output. The agent must handle unit semantics in the language layer.

**Impact:** No functional issue in arithmetic contexts, but worth noting when discussing the tool's scope in class.

---

### 🟡 Hosted Chat UI Is Not Production-Embeddable as-is
The hosted chat page is served by the n8n instance and requires the workflow to be active. Embedding it in an external site requires additional CORS configuration and a stable n8n URL.

**Impact:** Suitable for demos and internal use. Not a drop-in component for a customer-facing product.

**Mitigation:** Switch the Chat Trigger to `webhook` mode and build a custom frontend that posts to the webhook endpoint.

---

<a name="usecases-en"></a>
## Recommended Use Cases

**Well-suited for:**
- ✅ Classroom demonstration of the agent tool-use loop (ReAct pattern)
- ✅ First exposure to n8n's AI Agent node and subnode architecture
- ✅ Testing OpenAI credentials and chat UI setup before building more complex agents
- ✅ Comparing model behavior with and without a tool (remove the Calculator and observe)

**Recommended test inputs:**

| Input | Expected behavior |
|-------|-------------------|
| `¿Cuánto es el 18% de 3,750?` | Tool invoked |
| `Si tengo 5 productos a $240 con 10% de descuento, ¿cuánto pago?` | Multi-step reasoning + tool |
| `¿Cuánto es la raíz cuadrada de 1764?` | Tool invoked |
| `¿Cuál es la capital de Francia?` | Tool NOT invoked — model answers directly |

**Not suited for:**
- ❌ Multi-turn conversations requiring context retention
- ❌ Queries that need live data (news, prices, facts) — the agent has no retrieval tool
- ❌ Production deployments without error handling and a persistent memory store

---

<a name="extensions-en"></a>
## Extension Ideas

| Enhancement | Approach |
|---|---|
| Add session memory | Connect `memoryBufferWindow` as a subnode; enable `loadPreviousSession: memory` on the trigger |
| Add a second tool | Include Wikipedia or a weather API tool — forces the agent to choose between tools |
| Expose intermediate steps | Set `returnIntermediateSteps: true` to show the full reasoning chain in the output |
| Swap the model | Replace `gpt-4o-mini` with `gpt-4o` or a local model via Ollama to compare tool-use reliability |
| Add a Code tool | Replace the Calculator with a Code tool executing Python — broader scope, less safe |
| Multilingual system prompt | Remove `Responde siempre en espanol` and observe how the agent mirrors the user's language |

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
6. [Decisiones de diseño](#diseno-es)
7. [Limitaciones y advertencias](#limitaciones-es)
8. [Casos de uso recomendados](#casos-es)
9. [Ideas de extensión](#extensiones-es)

---

<a name="descripcion-es"></a>
## Descripción general

Este workflow expone un agente de IA mínimo a través de la interfaz de chat integrada en n8n. El agente tiene acceso a una sola herramienta — una Calculadora — y debe decidir en cada turno si invocarla o responder directamente desde su modelo de lenguaje.

El pipeline es intencionalmente simple:

```
[Chat Trigger] → [Agente IA] ← [OpenAI Chat Model]
                             ← [Calculator Tool]
```

> **Nota sobre credenciales:** El workflow viene con un placeholder `newCredential('OpenAI API')`. Debes conectar tu propia credencial de OpenAI antes de que el workflow pueda ejecutarse — consulta [Requisitos](#requisitos-es) para más detalles.

---

<a name="arquitectura-es"></a>
## Arquitectura

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         agente-calculadora                                │
│                                                                            │
│  ┌─────────────────┐                                                      │
│  │ Chat Trigger    │                                                      │
│  │ (UI integrada)  │                                                      │
│  └────────┬────────┘                                                      │
│           │  chatInput                                                    │
│           ▼                                                               │
│  ┌────────────────────────────────────────────────────┐                  │
│  │                   Agente IA (ReAct)                 │                  │
│  │                                                     │                  │
│  │  ┌──────────────────────┐  ┌──────────────────┐   │                  │
│  │  │  OpenAI Chat Model   │  │  Calculator Tool  │   │                  │
│  │  │  (gpt-4o-mini)       │  │  (sin creds.)     │   │                  │
│  │  └──────────────────────┘  └──────────────────┘   │                  │
│  └────────────────────────────────────────────────────┘                  │
│           │  output                                                       │
│           ▼                                                               │
│       [Respuesta en chat]                                                 │
└──────────────────────────────────────────────────────────────────────────┘
```

---

<a name="nodos-es"></a>
## Descripción de nodos

### 1. `Chat`
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.chatTrigger` v1.4 |
| Modo | Hosted Chat (UI servida por n8n) |
| Manejo de sesión | Automático — cada pestaña del navegador es una sesión aislada |
| Salida | String `chatInput` que se pasa al agente |

Expone la página de chat integrada en n8n y dispara el workflow en cada envío de mensaje. No requiere infraestructura externa — la UI es servida directamente por la instancia de n8n.

> **Nota:** La URL del chat está ligada al workflow activo. El workflow debe estar activado (no solo guardado) para que el endpoint de chat sea accesible.

---

### 2. `Agente Asistente`
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.agent` v3.1 |
| Fuente del prompt | `auto` — lee `chatInput` del Chat Trigger |
| System message | Ver [Decisiones de diseño](#diseno-es) |
| Iteraciones máximas | 5 |
| Pasos intermedios | No expuestos (por defecto) |
| Subnodos | `OpenAI Chat Model`, `Calculator` |

Orquesta el loop ReAct (Razonar + Actuar). En cada turno, el modelo decide si invocar la herramienta Calculator o responder directamente. El límite de iteraciones (5) previene loops descontrolados con entradas ambiguas.

---

### 3. `OpenAI Chat Model`
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.lmChatOpenAi` v1.3 |
| Modelo | `gpt-4o-mini` (por defecto) |
| Credenciales | `openAiApi` (configurable por el usuario) |

Provee el backbone de modelo de lenguaje para el agente. Se elige `gpt-4o-mini` por eficiencia de costo en un contexto de clase. Puede intercambiarse por cualquier modelo disponible en la API de OpenAI.

---

### 4. `Calculator`
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.toolCalculator` v1 |
| Credenciales | Ninguna requerida |
| Alcance | Solo expresiones aritméticas |

Evalúa expresiones matemáticas que le pasa el agente. Corre completamente dentro de n8n — sin llamada a API externa, sin cuota. El agente formula una expresión (ej. `2400 * 0.15`) y la herramienta devuelve el resultado numérico.

> **Nota:** La Calculator evalúa expresiones, no lenguaje natural. El agente es responsable de traducir la intención del usuario a una expresión aritmética válida antes de invocar la herramienta.

---

<a name="requisitos-es"></a>
## Requisitos y credenciales

### Credenciales requeridas en n8n

| Nombre de credencial | Tipo | Usada por | Cómo configurar |
|---|---|---|---|
| OpenAI API | `openAiApi` | `OpenAI Chat Model` | [Docs OpenAI en n8n](https://docs.n8n.io/integrations/builtin/credentials/openai/) |

### Pasos de configuración

1. Importar el workflow en tu instancia de n8n.
2. Abrir el nodo **OpenAI Chat Model** y conectar tu credencial de OpenAI.
3. (Opcional) Editar el system message en **Agente Asistente → Opciones → System Message**.
4. Activar el workflow.
5. Hacer clic en el botón **Chat** de la barra de n8n para abrir la UI integrada.

---

<a name="specs-es"></a>
## Especificaciones técnicas

| Componente | Tecnología |
|---|---|
| Motor de workflow | n8n |
| Trigger | Chat Trigger (UI integrada) |
| Tipo de agente | ReAct (Razonar + Actuar) |
| Modelo de lenguaje | OpenAI `gpt-4o-mini` |
| Herramientas | Calculator (nativo en n8n) |
| Memoria | Ninguna — sin estado entre mensajes |
| Credenciales requeridas | OpenAI API |

---

<a name="diseno-es"></a>
## Decisiones de diseño

### ¿Por qué sin memoria?
Este workflow es deliberadamente sin estado. Agregar memoria introduce un segundo concepto (persistencia de sesión) que compite por la atención con el objetivo principal: hacer visible el loop de uso de herramientas. La memoria se cubre en el workflow complementario `agente-investigador`.

### ¿Por qué Calculator y no una Code tool?
La herramienta Calculator tiene un alcance fijo y bien entendido — aritmética — lo que hace que el límite de decisión del agente sea fácil de razonar. Una Code tool puede ejecutar lógica arbitraria, lo que difumina la claridad conceptual necesaria en un ejemplo de primera sesión.

### ¿Por qué `promptType: auto`?
Con `auto`, el agente lee `chatInput` directamente del output del Chat Trigger sin requerir una expresión explícita. Esto mantiene la configuración del nodo mínima y reduce la superficie de errores de configuración durante una sesión en vivo.

### Razonamiento del system message
```
Eres un asistente util y amigable. Tienes acceso a una calculadora
para resolver operaciones matematicas. Responde siempre en espanol.
```
El prompt nombra explícitamente la herramienta disponible. Esto es importante: sin una mención explícita, los modelos más pequeños pueden subutilizar la herramienta en problemas que pueden resolver parcialmente por sí mismos — produciendo resultados menos precisos mientras aparentan tener éxito.

### ¿Por qué `maxIterations: 5`?
El valor por defecto (10) es generoso para un agente de calculadora simple. Cinco iteraciones son suficientes para cualquier problema aritmético alcanzable con esta herramienta, mientras se limita el costo de cómputo y se previenen loops en entradas patológicas.

---

<a name="limitaciones-es"></a>
## Limitaciones y advertencias

### 🔴 Sin memoria de sesión
Cada mensaje se procesa de forma independiente. El agente no tiene conciencia de los turnos anteriores en la misma conversación.

**Impacto:** El agente no puede responder preguntas de seguimiento que referencien contexto previo. Un usuario que pregunte "¿cuál fue ese resultado?" recibirá una respuesta basada únicamente en el mensaje actual.

**Mitigación:** Agregar un subnodo `memoryBufferWindow` al agente y establecer `loadPreviousSession: memory` en el Chat Trigger. Esta es la arquitectura usada en `agente-investigador`.

---

### 🟡 El modelo puede omitir la herramienta en expresiones simples
`gpt-4o-mini` ocasionalmente evalúa aritmética trivial (ej. `2 + 2`) sin invocar la Calculator. Esto no es un fallo — el modelo es capaz de aritmética mental simple — pero reduce la claridad del demo.

**Impacto:** Los alumnos pueden no ver el loop de uso de herramientas en todas las consultas.

**Mitigación:** Usar entradas claramente no triviales, como porcentajes, expresiones multi-paso o raíces cuadradas. Ver [inputs de prueba recomendados](#casos-es).

---

### 🟡 La Calculator no tiene conciencia de unidades ni contexto
La herramienta evalúa expresiones como números puros. `15% de 2400 pesos` y `15% de 2400 km` producen el mismo output. El agente debe manejar la semántica de unidades en la capa de lenguaje.

**Impacto:** Sin problema funcional en contextos aritméticos, pero vale la pena señalarlo al discutir el alcance de la herramienta en clase.

---

### 🟡 La UI de chat integrada no es embebible en producción tal cual
La página de chat está servida por la instancia de n8n y requiere que el workflow esté activo. Embebida en un sitio externo requiere configuración adicional de CORS y una URL estable de n8n.

**Impacto:** Adecuada para demos y uso interno. No es un componente listo para un producto de cara al cliente.

**Mitigación:** Cambiar el Chat Trigger a modo `webhook` y construir un frontend personalizado que haga POST al endpoint del webhook.

---

<a name="casos-es"></a>
## Casos de uso recomendados

**Adecuado para:**
- ✅ Demostración en clase del loop de uso de herramientas (patrón ReAct)
- ✅ Primera exposición al nodo AI Agent de n8n y su arquitectura de subnodos
- ✅ Probar credenciales de OpenAI y la configuración de la UI de chat antes de construir agentes más complejos
- ✅ Comparar el comportamiento del modelo con y sin herramienta (quitar la Calculator y observar)

**Inputs de prueba recomendados:**

| Input | Comportamiento esperado |
|-------|------------------------|
| `¿Cuánto es el 18% de 3,750?` | Herramienta invocada |
| `Si tengo 5 productos a $240 con 10% de descuento, ¿cuánto pago?` | Razonamiento multi-paso + herramienta |
| `¿Cuánto es la raíz cuadrada de 1764?` | Herramienta invocada |
| `¿Cuál es la capital de Francia?` | Herramienta NO invocada — modelo responde directamente |

**No adecuado para:**
- ❌ Conversaciones multi-turno que requieren retención de contexto
- ❌ Consultas que necesitan datos en tiempo real (noticias, precios, hechos) — el agente no tiene herramienta de recuperación
- ❌ Despliegues en producción sin manejo de errores y un almacén de memoria persistente

---

<a name="extensiones-es"></a>
## Ideas de extensión

| Mejora | Enfoque |
|---|---|
| Agregar memoria de sesión | Conectar `memoryBufferWindow` como subnodo; habilitar `loadPreviousSession: memory` en el trigger |
| Agregar una segunda herramienta | Incluir Wikipedia o una API del clima — obliga al agente a elegir entre herramientas |
| Exponer pasos intermedios | Establecer `returnIntermediateSteps: true` para mostrar la cadena de razonamiento completa en el output |
| Cambiar el modelo | Reemplazar `gpt-4o-mini` por `gpt-4o` o un modelo local vía Ollama para comparar la fiabilidad del uso de herramientas |
| Agregar una Code tool | Reemplazar la Calculator por una Code tool que ejecute Python — mayor alcance, menos segura |
| System prompt multilingüe | Eliminar `Responde siempre en espanol` y observar cómo el agente refleja el idioma del usuario |

---

## Estructura de archivos

```
n8n-agent-flows/
└── agents/
    └── agente-calculadora/
        ├── agente-calculadora.json     # Exportable n8n workflow / Workflow exportable de n8n
        └── README.md                   # This file / Este archivo
```

---

*Part of the [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) repository — a collection of n8n workflows for AI systems, RAG pipelines, and process automation.*

*Parte del repositorio [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) — una colección de workflows de n8n para sistemas AI, pipelines RAG y automatización de procesos.*