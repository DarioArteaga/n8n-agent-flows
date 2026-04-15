# agente-investigador

> **n8n workflow** — Conversational research agent: receives a user message through n8n's built-in chat UI, searches Wikipedia when factual retrieval is needed, and maintains a short-term memory window across turns in the same session. Designed as a classroom demonstration of tool use combined with conversational context.

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

This workflow exposes an AI agent through n8n's hosted chat interface. The agent has access to Wikipedia as a live knowledge source and a session memory window that persists context across turns within the same conversation. On each message, the agent reasons about whether the query requires factual retrieval, whether it can be answered from memory, or whether the model's own knowledge is sufficient.

The pipeline adds two subnodes to the minimal agent pattern:

```
[Chat Trigger] → [AI Agent] ← [OpenAI Chat Model]
                            ← [Wikipedia Tool]
                            ← [Simple Memory]
```

> **Note on credentials:** The workflow ships with a `newCredential('OpenAI API')` placeholder. You must connect your own OpenAI credential before the workflow will run — see [Prerequisites](#prerequisites-en) for details.

---

<a name="architecture-en"></a>
## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        agente-investigador                                │
│                                                                            │
│  ┌─────────────────┐                                                      │
│  │ Chat Trigger    │                                                      │
│  │ (hosted UI)     │                                                      │
│  └────────┬────────┘                                                      │
│           │  chatInput + sessionId                                        │
│           ▼                                                               │
│  ┌────────────────────────────────────────────────────────────┐          │
│  │                      AI Agent (ReAct)                       │          │
│  │                                                             │          │
│  │  ┌──────────────────┐ ┌────────────────┐ ┌─────────────┐  │          │
│  │  │ OpenAI Chat Model│ │ Wikipedia Tool │ │Simple Memory│  │          │
│  │  │ (gpt-4o-mini)    │ │ (no creds.)    │ │(window = 5) │  │          │
│  │  └──────────────────┘ └────────────────┘ └─────────────┘  │          │
│  └────────────────────────────────────────────────────────────┘          │
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
| Session handling | `loadPreviousSession: memory` — restores context from the memory node |
| Output | `chatInput` string + `sessionId` passed to agent and memory |

Exposes n8n's built-in chat page and fires the workflow on each message submission. The `loadPreviousSession: memory` setting instructs the Chat Trigger to coordinate session identity with the attached memory subnode — each browser tab maintains its own isolated conversation context.

> **Note:** The workflow must be activated (not just saved) for the chat endpoint to be reachable.

---

### 2. `Agente Investigador`
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.agent` v3.1 |
| Prompt source | `auto` — reads `chatInput` from the Chat Trigger |
| System message | See [Design Decisions](#design-en) |
| Max iterations | 5 |
| Intermediate steps | Not exposed (default) |
| Subnodes | `OpenAI Chat Model`, `Wikipedia`, `Memoria de Sesión` |

Orchestrates the ReAct loop with access to two resources: a retrieval tool (Wikipedia) and a memory buffer (past conversation turns). On each turn, the model decides whether to search Wikipedia, consult memory, or answer from its own knowledge.

---

### 3. `OpenAI Chat Model`
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.lmChatOpenAi` v1.3 |
| Model | `gpt-4o-mini` (default) |
| Credentials | `openAiApi` (user-configured) |

Provides the language model backbone for the agent. Can be swapped for any OpenAI-compatible model.

---

### 4. `Wikipedia`
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.toolWikipedia` v1 |
| Credentials | None required |
| Output | Article summary for the matched topic |

Searches Wikipedia and returns the summary section of the most relevant article. The agent formulates a search query from the user's message and passes it to the tool. No API key or rate limit applies — the tool uses Wikipedia's public API.

> **Note:** Wikipedia returns summaries, not full articles. For niche topics or disambiguation cases, the returned content may be shallow or off-target. The agent does not validate the retrieved content — it trusts the summary as-is.

---

### 5. `Memoria de Sesión`
| Property | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.memoryBufferWindow` v1.3 |
| Session ID source | `fromInput` — reads `sessionId` from the Chat Trigger |
| Context window | 5 exchanges (configurable) |
| Storage | n8n process memory — no external service |

Stores the last N conversation exchanges and injects them into the agent's context on each turn. The `fromInput` session type ensures that each chat session (identified by the trigger's `sessionId`) has its own isolated memory window — conversations between different users or browser tabs do not bleed into each other.

> **Note:** Memory is held in n8n's process memory. It does not persist across workflow restarts or n8n service updates. For production scenarios, replace with a Postgres-backed memory node.

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
3. (Optional) Adjust the context window size in **Memoria de Sesión → Context Window Length** (default: 5).
4. (Optional) Edit the system message in **Agente Investigador → Options → System Message**.
5. Activate the workflow.
6. Click the **Chat** button in the n8n toolbar to open the hosted UI.

---

<a name="specs-en"></a>
## Technical Specs

| Component | Technology |
|---|---|
| Workflow engine | n8n |
| Trigger | Chat Trigger (hosted UI) |
| Agent type | ReAct (Reason + Act) |
| Language model | OpenAI `gpt-4o-mini` |
| Tools | Wikipedia (n8n built-in) |
| Memory | `memoryBufferWindow` — 5-exchange window, process-scoped |
| Credentials required | OpenAI API |

---

<a name="design-en"></a>
## Design Decisions

### Why Wikipedia and not a web search tool?
Wikipedia provides structured, consistent summaries with predictable output format. A general web search tool would return noisier, less consistent results that introduce retrieval quality as a variable — which competes with the session in making tool use and memory the focal concepts. Wikipedia keeps the demo clean.

### Why `sessionIdType: fromInput`?
The Chat Trigger automatically emits a `sessionId` field per browser session. Using `fromInput` binds the memory window to that identifier without requiring any manual configuration. The result is automatic session isolation: each user or browser tab gets its own conversation context with no additional setup.

### Why a 5-exchange context window?
Five exchanges cover enough conversational depth to demonstrate memory working (follow-up questions, pronoun resolution, topic continuity) without consuming excessive tokens on each API call. In a classroom session, conversations rarely exceed 5–8 turns before a topic reset.

### System message rationale
```
Eres un asistente investigador. Puedes buscar informacion en Wikipedia
para responder preguntas. Siempre cita la fuente cuando uses Wikipedia.
Responde en espanol de forma clara y concisa.
```
The instruction to cite the source when using Wikipedia serves a pedagogical purpose: it makes the tool invocation visible in the output. Students can see "according to Wikipedia" in the response and trace it back to a tool call in the execution log.

### Why not include the Calculator tool here?
Adding the Calculator would expand the agent's decision space and shift the session's focus from memory and retrieval to multi-tool routing. Keeping tools separated across two workflows makes each concept independently demonstrable.

---

<a name="limitations-en"></a>
## Limitations & Warnings

### 🔴 Memory Does Not Persist Across Restarts
`memoryBufferWindow` stores conversation context in n8n's process memory. Restarting n8n, updating the workflow, or redeploying the instance clears all active session memory silently.

**Impact:** Any ongoing conversation is lost on restart. This is not visible to the user — the agent simply loses context without warning.

**Mitigation:** For persistent memory, replace `memoryBufferWindow` with a Postgres-backed memory node (`memoryPostgresChat`) or a Redis-backed equivalent. Both require additional credential setup but survive restarts.

---

### 🔴 Wikipedia Content Is Not Validated
The agent trusts Wikipedia's returned summary without any verification step. If the search query is ambiguous or the topic is contested, the agent may present inaccurate or outdated information as fact.

**Impact:** Factually incorrect responses with confident framing. In a classroom context this is actually useful — it demonstrates why retrieval alone does not solve hallucination; source quality matters.

**Mitigation:** For production use, add a grounding step that validates retrieved content against a curated knowledge base before passing it to the agent.

---

### 🟡 Context Window Silently Drops Older Turns
When the conversation exceeds 5 exchanges, the oldest turns are dropped from the context window without any signal to the user or the agent. The agent loses awareness of early conversation content.

**Impact:** The agent may appear inconsistent or forgetful on long conversations — answering as if earlier context never existed.

**Mitigation:** Increase `contextWindowLength` for longer sessions, or implement a summarization step that compresses older turns into a running summary before they drop off.

---

### 🟡 Wikipedia Returns Summaries, Not Full Articles
For niche, technical, or recently updated topics, the summary section may be too shallow to answer the user's question adequately. The agent has no way to request deeper article content through this tool.

**Impact:** Incomplete answers on specialized topics, with no indication that more content exists.

**Mitigation:** Replace or supplement with a tool that retrieves full article sections (e.g., via the Wikipedia REST API with a Code node), or add a web search tool for broader coverage.

---

### 🟡 Hosted Chat UI Is Not Production-Embeddable as-is
The hosted chat page is served by the n8n instance and requires the workflow to be active. Embedding it in an external site requires additional CORS configuration and a stable n8n URL.

**Impact:** Suitable for demos and internal use. Not a drop-in component for a customer-facing product.

**Mitigation:** Switch the Chat Trigger to `webhook` mode and build a custom frontend that posts to the webhook endpoint.

---

<a name="usecases-en"></a>
## Recommended Use Cases

**Well-suited for:**
- ✅ Classroom demonstration of tool use + session memory in a single workflow
- ✅ Showing how agents retrieve external knowledge rather than hallucinate facts
- ✅ Making memory visible through follow-up questions that reference prior context
- ✅ Comparing behavior with and without the memory node (remove it and repeat the sequence)

**Recommended test sequence:**

| Turn | Input | What it demonstrates |
|------|-------|---------------------|
| 1 | `¿Quién fue Alan Turing?` | Wikipedia invoked for factual retrieval |
| 2 | `¿Y qué relación tiene con lo que me acabas de decir?` | Memory in use — agent references the previous answer |
| 3 | `¿Cuándo nació?` | Memory + Wikipedia disambiguation |
| 4 | `¿Cuál es la capital de Australia?` | Tool may or may not be invoked — good discussion point |

Turn 2 is the key classroom moment. Remove the memory node, repeat the sequence, and the agent will fail to connect turn 2 to turn 1 — making the memory component's role concrete.

**Not suited for:**
- ❌ Long-running conversations where context from early turns must be retained beyond the window size
- ❌ Topics requiring deep or real-time factual accuracy (news, prices, recent events)
- ❌ Production deployments without a persistent memory backend and error handling

---

<a name="extensions-en"></a>
## Extension Ideas

| Enhancement | Approach |
|---|---|
| Persistent memory | Replace `memoryBufferWindow` with `memoryPostgresChat` — survives restarts, queryable |
| Add the Calculator | Include the Calculator tool alongside Wikipedia to handle hybrid queries |
| Expose intermediate steps | Set `returnIntermediateSteps: true` to show which tool was invoked and why |
| Memory summarization | Add a Code node that compresses older turns into a summary before they drop off the window |
| Multi-language retrieval | Query Wikipedia in the user's language by dynamically setting the `language` parameter |
| Source citation in metadata | Extract Wikipedia article URL from the tool response and append it to the agent's reply |

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

Este workflow expone un agente de IA a través de la interfaz de chat integrada en n8n. El agente tiene acceso a Wikipedia como fuente de conocimiento en tiempo real y una ventana de memoria de sesión que persiste el contexto entre turnos dentro de la misma conversación. En cada mensaje, el agente razona si la consulta requiere recuperación factual, si puede responderse desde la memoria, o si el conocimiento propio del modelo es suficiente.

El pipeline agrega dos subnodos al patrón de agente mínimo:

```
[Chat Trigger] → [Agente IA] ← [OpenAI Chat Model]
                             ← [Wikipedia Tool]
                             ← [Simple Memory]
```

> **Nota sobre credenciales:** El workflow viene con un placeholder `newCredential('OpenAI API')`. Debes conectar tu propia credencial de OpenAI antes de que el workflow pueda ejecutarse — consulta [Requisitos](#requisitos-es) para más detalles.

---

<a name="arquitectura-es"></a>
## Arquitectura

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        agente-investigador                                │
│                                                                            │
│  ┌─────────────────┐                                                      │
│  │ Chat Trigger    │                                                      │
│  │ (UI integrada)  │                                                      │
│  └────────┬────────┘                                                      │
│           │  chatInput + sessionId                                        │
│           ▼                                                               │
│  ┌────────────────────────────────────────────────────────────┐          │
│  │                      Agente IA (ReAct)                      │          │
│  │                                                             │          │
│  │  ┌──────────────────┐ ┌────────────────┐ ┌─────────────┐  │          │
│  │  │ OpenAI Chat Model│ │ Wikipedia Tool │ │Memoria Ses. │  │          │
│  │  │ (gpt-4o-mini)    │ │ (sin creds.)   │ │(ventana = 5)│  │          │
│  │  └──────────────────┘ └────────────────┘ └─────────────┘  │          │
│  └────────────────────────────────────────────────────────────┘          │
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
| Manejo de sesión | `loadPreviousSession: memory` — restaura contexto desde el nodo de memoria |
| Salida | String `chatInput` + `sessionId` pasados al agente y a la memoria |

Expone la página de chat integrada en n8n y dispara el workflow en cada envío de mensaje. La configuración `loadPreviousSession: memory` instruye al Chat Trigger a coordinar la identidad de sesión con el subnodo de memoria adjunto — cada pestaña del navegador mantiene su propio contexto de conversación aislado.

> **Nota:** El workflow debe estar activado (no solo guardado) para que el endpoint de chat sea accesible.

---

### 2. `Agente Investigador`
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.agent` v3.1 |
| Fuente del prompt | `auto` — lee `chatInput` del Chat Trigger |
| System message | Ver [Decisiones de diseño](#diseno-es) |
| Iteraciones máximas | 5 |
| Pasos intermedios | No expuestos (por defecto) |
| Subnodos | `OpenAI Chat Model`, `Wikipedia`, `Memoria de Sesión` |

Orquesta el loop ReAct con acceso a dos recursos: una herramienta de recuperación (Wikipedia) y un buffer de memoria (turnos pasados de la conversación). En cada turno, el modelo decide si buscar en Wikipedia, consultar la memoria, o responder desde su propio conocimiento.

---

### 3. `OpenAI Chat Model`
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.lmChatOpenAi` v1.3 |
| Modelo | `gpt-4o-mini` (por defecto) |
| Credenciales | `openAiApi` (configurable por el usuario) |

Provee el backbone de modelo de lenguaje para el agente. Puede intercambiarse por cualquier modelo compatible con la API de OpenAI.

---

### 4. `Wikipedia`
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.toolWikipedia` v1 |
| Credenciales | Ninguna requerida |
| Salida | Resumen del artículo para el tema encontrado |

Busca en Wikipedia y devuelve la sección de resumen del artículo más relevante. El agente formula una consulta de búsqueda a partir del mensaje del usuario y la pasa a la herramienta. No aplica API key ni límite de tasa — la herramienta usa la API pública de Wikipedia.

> **Nota:** Wikipedia devuelve resúmenes, no artículos completos. Para temas de nicho o casos de desambiguación, el contenido devuelto puede ser superficial o fuera de tema. El agente no valida el contenido recuperado — confía en el resumen tal cual.

---

### 5. `Memoria de Sesión`
| Propiedad | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.memoryBufferWindow` v1.3 |
| Fuente del ID de sesión | `fromInput` — lee `sessionId` del Chat Trigger |
| Ventana de contexto | 5 intercambios (configurable) |
| Almacenamiento | Memoria del proceso de n8n — sin servicio externo |

Almacena los últimos N intercambios de la conversación y los inyecta en el contexto del agente en cada turno. El tipo de sesión `fromInput` asegura que cada sesión de chat (identificada por el `sessionId` del trigger) tenga su propia ventana de memoria aislada — las conversaciones entre diferentes usuarios o pestañas del navegador no se mezclan entre sí.

> **Nota:** La memoria se mantiene en la memoria del proceso de n8n. No persiste entre reinicios del workflow o actualizaciones del servicio de n8n. Para escenarios de producción, reemplazar por un nodo de memoria respaldado por Postgres.

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
3. (Opcional) Ajustar el tamaño de la ventana en **Memoria de Sesión → Context Window Length** (por defecto: 5).
4. (Opcional) Editar el system message en **Agente Investigador → Opciones → System Message**.
5. Activar el workflow.
6. Hacer clic en el botón **Chat** de la barra de n8n para abrir la UI integrada.

---

<a name="specs-es"></a>
## Especificaciones técnicas

| Componente | Tecnología |
|---|---|
| Motor de workflow | n8n |
| Trigger | Chat Trigger (UI integrada) |
| Tipo de agente | ReAct (Razonar + Actuar) |
| Modelo de lenguaje | OpenAI `gpt-4o-mini` |
| Herramientas | Wikipedia (nativo en n8n) |
| Memoria | `memoryBufferWindow` — ventana de 5 intercambios, scoped al proceso |
| Credenciales requeridas | OpenAI API |

---

<a name="diseno-es"></a>
## Decisiones de diseño

### ¿Por qué Wikipedia y no una herramienta de búsqueda web?
Wikipedia provee resúmenes estructurados y consistentes con un formato de salida predecible. Una herramienta de búsqueda web general devolvería resultados más ruidosos e inconsistentes que introducen la calidad de recuperación como una variable — lo que compite con hacer del uso de herramientas y la memoria los conceptos focales. Wikipedia mantiene el demo limpio.

### ¿Por qué `sessionIdType: fromInput`?
El Chat Trigger emite automáticamente un campo `sessionId` por sesión del navegador. Usar `fromInput` vincula la ventana de memoria a ese identificador sin requerir ninguna configuración manual. El resultado es aislamiento automático de sesiones: cada usuario o pestaña del navegador obtiene su propio contexto de conversación sin configuración adicional.

### ¿Por qué una ventana de contexto de 5 intercambios?
Cinco intercambios cubren suficiente profundidad conversacional para demostrar que la memoria funciona (preguntas de seguimiento, resolución de pronombres, continuidad de tema) sin consumir tokens excesivos en cada llamada a la API. En una sesión de clase, las conversaciones raramente superan los 5–8 turnos antes de un cambio de tema.

### Razonamiento del system message
```
Eres un asistente investigador. Puedes buscar informacion en Wikipedia
para responder preguntas. Siempre cita la fuente cuando uses Wikipedia.
Responde en espanol de forma clara y concisa.
```
La instrucción de citar la fuente cuando se usa Wikipedia tiene un propósito pedagógico: hace visible la invocación de la herramienta en el output. Los alumnos pueden ver "según Wikipedia" en la respuesta y rastrearlo hasta una llamada de herramienta en el log de ejecución.

### ¿Por qué no incluir la herramienta Calculator aquí?
Agregar la Calculator expandiría el espacio de decisión del agente y desplazaría el enfoque de la sesión desde la memoria y la recuperación hacia el enrutamiento multi-herramienta. Mantener las herramientas separadas entre dos workflows hace que cada concepto sea demostrable de forma independiente.

---

<a name="limitaciones-es"></a>
## Limitaciones y advertencias

### 🔴 La memoria no persiste entre reinicios
`memoryBufferWindow` almacena el contexto de conversación en la memoria del proceso de n8n. Reiniciar n8n, actualizar el workflow o redesplegar la instancia borra toda la memoria de sesión activa de forma silenciosa.

**Impacto:** Cualquier conversación en curso se pierde al reiniciar. Esto no es visible para el usuario — el agente simplemente pierde contexto sin advertencia.

**Mitigación:** Para memoria persistente, reemplazar `memoryBufferWindow` por un nodo de memoria respaldado por Postgres (`memoryPostgresChat`) o un equivalente con Redis. Ambos requieren configuración adicional de credenciales pero sobreviven a los reinicios.

---

### 🔴 El contenido de Wikipedia no se valida
El agente confía en el resumen devuelto por Wikipedia sin ningún paso de verificación. Si la consulta de búsqueda es ambigua o el tema es controvertido, el agente puede presentar información inexacta u obsoleta como un hecho.

**Impacto:** Respuestas factualmente incorrectas con formulación confiada. En un contexto de clase esto es útil — demuestra por qué la recuperación por sí sola no resuelve la alucinación; la calidad de la fuente importa.

**Mitigación:** Para uso en producción, agregar un paso de grounding que valide el contenido recuperado contra una base de conocimiento curada antes de pasarlo al agente.

---

### 🟡 La ventana de contexto descarta turnos antiguos silenciosamente
Cuando la conversación supera los 5 intercambios, los turnos más antiguos se descartan de la ventana de contexto sin ninguna señal al usuario ni al agente. El agente pierde conciencia del contenido temprano de la conversación.

**Impacto:** El agente puede parecer inconsistente o olvidadizo en conversaciones largas — respondiendo como si el contexto anterior no hubiera existido.

**Mitigación:** Aumentar `contextWindowLength` para sesiones más largas, o implementar un paso de resumen que comprima los turnos más antiguos en un resumen acumulado antes de que salgan de la ventana.

---

### 🟡 Wikipedia devuelve resúmenes, no artículos completos
Para temas de nicho, técnicos o recientemente actualizados, la sección de resumen puede ser demasiado superficial para responder adecuadamente la pregunta del usuario. El agente no tiene forma de solicitar contenido más profundo del artículo a través de esta herramienta.

**Impacto:** Respuestas incompletas en temas especializados, sin indicación de que existe más contenido.

**Mitigación:** Reemplazar o complementar con una herramienta que recupere secciones completas del artículo (ej. mediante la API REST de Wikipedia con un nodo Code), o agregar una herramienta de búsqueda web para mayor cobertura.

---

### 🟡 La UI de chat integrada no es embebible en producción tal cual
La página de chat está servida por la instancia de n8n y requiere que el workflow esté activo. Embebida en un sitio externo requiere configuración adicional de CORS y una URL estable de n8n.

**Impacto:** Adecuada para demos y uso interno. No es un componente listo para un producto de cara al cliente.

**Mitigación:** Cambiar el Chat Trigger a modo `webhook` y construir un frontend personalizado que haga POST al endpoint del webhook.

---

<a name="casos-es"></a>
## Casos de uso recomendados

**Adecuado para:**
- ✅ Demostración en clase de uso de herramientas + memoria de sesión en un solo workflow
- ✅ Mostrar cómo los agentes recuperan conocimiento externo en lugar de alucinar hechos
- ✅ Hacer visible la memoria mediante preguntas de seguimiento que referencian contexto previo
- ✅ Comparar comportamiento con y sin el nodo de memoria (quitarlo y repetir la secuencia)

**Secuencia de prueba recomendada:**

| Turno | Input | Qué demuestra |
|-------|-------|---------------|
| 1 | `¿Quién fue Alan Turing?` | Wikipedia invocada para recuperación factual |
| 2 | `¿Y qué relación tiene con lo que me acabas de decir?` | Memoria en uso — el agente referencia la respuesta anterior |
| 3 | `¿Cuándo nació?` | Memoria + desambiguación con Wikipedia |
| 4 | `¿Cuál es la capital de Australia?` | La herramienta puede o no invocarse — buen punto de discusión |

El turno 2 es el momento clave de la clase. Quitar el nodo de memoria, repetir la secuencia, y el agente fallará en conectar el turno 2 con el turno 1 — haciendo concreto el rol del componente de memoria.

**No adecuado para:**
- ❌ Conversaciones largas donde el contexto de los primeros turnos debe retenerse más allá del tamaño de la ventana
- ❌ Temas que requieren precisión factual profunda o en tiempo real (noticias, precios, eventos recientes)
- ❌ Despliegues en producción sin un backend de memoria persistente y manejo de errores

---

<a name="extensiones-es"></a>
## Ideas de extensión

| Mejora | Enfoque |
|---|---|
| Memoria persistente | Reemplazar `memoryBufferWindow` por `memoryPostgresChat` — sobrevive reinicios, consultable |
| Agregar la Calculator | Incluir la herramienta Calculator junto a Wikipedia para manejar consultas híbridas |
| Exponer pasos intermedios | Establecer `returnIntermediateSteps: true` para mostrar qué herramienta fue invocada y por qué |
| Resumen de memoria | Agregar nodo Code que comprima turnos anteriores en un resumen antes de que salgan de la ventana |
| Recuperación multilingüe | Consultar Wikipedia en el idioma del usuario estableciendo dinámicamente el parámetro `language` |
| Cita de fuente en metadatos | Extraer la URL del artículo de Wikipedia de la respuesta de la herramienta y agregarla a la respuesta del agente |

---

## Estructura de archivos

```
n8n-agent-flows/
└── agents/
    └── agente-investigador/
        ├── agente-investigador.json     # Exportable n8n workflow / Workflow exportable de n8n
        └── README.md                    # This file / Este archivo
```

---

*Part of the [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) repository — a collection of n8n workflows for AI systems, RAG pipelines, and process automation.*

*Parte del repositorio [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) — una colección de workflows de n8n para sistemas AI, pipelines RAG y automatización de procesos.*