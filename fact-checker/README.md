# Fact Checker — Multi-Agent News Verification System

> **n8n workflow** — Verification-and-Contrast architecture: an orchestrator coordinates three specialist agents (press tracker + data investigator + logic auditor) to audit the veracity of a news item or claim and issue a confidence-scored verdict.

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

This workflow implements a **Verification-and-Contrast architecture** where a central orchestrator (Claude Sonnet 4.5) coordinates three specialist agents to audit the veracity of a news item or claim.

The investigation pipeline runs as follows:
1. `agente_serp` tracks how media outlets are reporting the news
2. `agente_tavily` searches for factual evidence in official sources
3. `agente_verificador` cross-references both reports and issues a verdict with a confidence level

---

<a name="architecture-en"></a>
## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         LangChain Sub-graph                           │
│                                                                       │
│  Chat Input                                                           │
│      │                                                                │
│      ▼                                                                │
│  orquestador_central ◄──── Sonnet 4.5 (LLM)                         │
│      │                ◄──── windows_buffer_memory (10 turns)         │
│      │ (ai_tool ×3)                                                   │
│      ├──────────────► agente_serp       ◄── GPT-4o Mini + SerpAPI    │
│      ├──────────────► agente_tavily     ◄── GPT-4o     + Tavily      │
│      └──────────────► agente_verificador ◄── GPT-4o                  │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

> **Diagram note:** `ai_tool` connections belong to n8n's LangChain sub-graph. The orchestrator controls invocation order at runtime. Window memory persists across the chat session.

---

<a name="nodes-en"></a>
## Node Breakdown

| Node | Type | Model / Tool | Role |
|------|------|--------------|------|
| `When chat message received` | Chat Trigger | — | Entry point — receives news item or claim to verify |
| `orquestador_central` | AI Agent | Claude Sonnet 4.5 | Orchestrator — manages investigation plan, synthesizes final report |
| `windows_buffer_memory` | Memory Buffer Window | — | 10-turn context window for multi-turn investigations |
| `agente_serp` | Agent Tool | GPT-4o Mini + SerpAPI | Press tracker — maps current media coverage and source types |
| `agente_tavily` | Agent Tool | GPT-4o + Tavily | Data investigator — retrieves official statements and factual evidence |
| `agente_verificador` | Agent Tool | GPT-4o | Logic auditor — cross-references both reports and emits confidence verdict |
| `Sonnet 4.5` | LM Chat Anthropic | claude-sonnet-4-5 | LLM for orchestrator |
| `GPT 4o Mini` | LM Chat OpenAI | gpt-4o-mini | LLM for SERP agent |
| `GPT 4o` | LM Chat OpenAI | gpt-4o | Shared LLM for Tavily and verificador agents |
| `SerpAPI` | Tool SERP | — | Google search — maps media narrative |
| `Tavily` | Tool Tavily | — | Deep research — retrieves primary sources and official data |

### Agent System Prompts Summary

**`agente_serp`** (GPT-4o Mini + SerpAPI)
Press tracker. Identifies the 5 main sources reporting the news and classifies them into: Official media / Social networks / Opinion blogs.

**`agente_tavily`** (GPT-4o + Tavily)
Data investigator. Searches for factual evidence — statistics, official statements, research articles. Prioritizes technical accuracy over narrative.

**`agente_verificador`** (GPT-4o)
Independent logic auditor. Receives reports from `serp` and `tavily`, flags contradictions between media narrative and official data, and issues a confidence verdict: **Low / Medium / High**.

### Orchestrator Output Format

The `orquestador_central` delivers a structured report to the user:
1. The evaluated news item or claim
2. Media source analysis (report from `agente_serp`)
3. Factual evidence found (report from `agente_tavily`)
4. Detected contradictions
5. Final verdict with confidence level

---

<a name="design-en"></a>
## Design Decisions

### Why different models per agent

Model assignment is deliberate and reflects a cost/capacity strategy:

| Agent | Model | Justification |
|--------|--------|---------------|
| `orquestador_central` | Claude Sonnet 4.5 | Meta-reasoning, multi-source synthesis, final report drafting |
| `agente_serp` | GPT-4o Mini | Well-scoped task: classifying URLs and headlines. No deep reasoning required |
| `agente_tavily` | GPT-4o | Needs to interpret technical documents, official statements, research papers |
| `agente_verificador` | GPT-4o | Comparative logical reasoning — detecting contradictions requires analytical capacity |

> The model selection isn't default or arbitrary — it reflects a cost-quality trade-off at each layer of the pipeline. This is a design decision worth documenting explicitly.

### Why Window Buffer Memory at the orchestrator level, not in tools

Memory persists at the orchestrator level because it maintains the state of the investigation. Tools are stateless by design — they execute an atomic task and return their result. This simplifies each specialist's behavior and makes the investigation cumulative without complicating tool prompts.

### Why SerpAPI for press and Tavily for data

SerpAPI is better for mapping the current media ecosystem (what is being said, where, and with what tone). Tavily is optimized for retrieving primary source content with higher fidelity — ideal for factual evidence. Separating these responsibilities prevents a single tool from mixing narrative with data.

### Why the verificador is independent and not the orchestrator

If the orchestrator directly synthesized the reports, it would introduce its planning bias into the verdict. Delegating verification to an independent agent implements a **separation of powers principle** within the system — the judge is not the same as the investigator.

---

<a name="setup-en"></a>
## Setup & Credentials

| Credential | Node | Notes |
|-----------|------|-------|
| `anthropicApi` | Sonnet 4.5 | Anthropic API key |
| `openAiApi` | GPT 4o Mini, GPT 4o | OpenAI API key |
| SerpAPI key | SerpAPI tool node | Register at serpapi.com — free plan: 100 searches/month |
| Tavily API key | Tavily tool node | Register at tavily.com — free plan available |

🔴 **SerpAPI and Tavily are external services with quotas.** Monitor consumption in intensive use or large-scale evaluations. The flow has no built-in rate limit handling.

🟡 **Window memory is session-scoped** — it resets when starting a new conversation in the Chat Trigger. There is no persistence between sessions.

---

<a name="usage-en"></a>
## Usage Example

**Input:**
```
Verify this claim: "Mexico reached 100% renewable energy in 2024"
```

**Expected output structure:**
```
📋 CLAIM EVALUATED:
[User's exact claim]

📰 MEDIA SOURCE ANALYSIS:
[agente_serp report: who says it, source type]

🔬 FACTUAL EVIDENCE:
[agente_tavily report: official data, statements, statistics]

⚠️ DETECTED CONTRADICTIONS:
[List of discrepancies between narrative and data]

🏁 VERDICT: [Low / Medium / High confidence]
[Confidence level justification]
```

---

<a name="limitations-en"></a>
## Limitations & Known Issues

🔴 **The verdict is probabilistic, not deterministic.** `agente_verificador` issues a confidence level based on LLM reasoning, not a formal scoring function. For high-stakes applications, output must be reviewed by a human.

🔴 **SerpAPI returns Google Search results — subject to SEO and personalization.** Results may vary by region or language. `agente_serp` may receive biased coverage depending on the query.

🟡 **Tavily has a retrieval depth limit.** For news with very technical documents or paywalled sources, the agent may not access the full content.

🟡 **The flow does not persist reports.** Reports from each session are lost when the chat is closed. For audit or traceability, consider adding a write node to Google Sheets or Supabase at the end of the pipeline.

🟡 **No forced language in tools.** If input is in Spanish, tools tend to respond in Spanish, but this is not guaranteed. For language consistency, specify the output language in each system prompt.

---

<a name="didactic-en"></a>
## Didactic Notes

This flow illustrates architectural patterns relevant to verification systems and agents with multiple sources of truth:

1. **Source vs. data separation.** `agente_serp` maps the narrative (what is being said). `agente_tavily` searches for evidence (what can be verified). This epistemological distinction is the core of fact-checking and of any RAG system with multiple retrieval sources.

2. **Verification agent as independent contrast layer.** Instead of the orchestrator synthesizing directly, delegating verification to a third agent implements a separation of responsibilities that reduces confirmation bias in the system.

3. **Model sizing by task.** GPT-4o Mini for headline classification, GPT-4o for analytical reasoning, Claude for complex synthesis. This "right-sizing" pattern is directly applicable to production pipelines where cost per token is a real constraint.

4. **Memory at the right level.** The window memory lives in the orchestrator, not in tools. This allows the investigation to be cumulative without adding state to each specialist.

---

<a name="file-structure-en"></a>
## File Structure

```
n8n-agent-flows/
└── fact-checker/
    ├── fact-checker.json     # Exportable n8n workflow
    └── README.md             # This file
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

Este flujo implementa una **arquitectura de Verificación y Contraste** con un orquestador central (Claude Sonnet 4.5) que coordina tres agentes especialistas para auditar la veracidad de una noticia o afirmación.

El flujo de trabajo es:
1. `agente_serp` rastrea cómo los medios están reportando la noticia
2. `agente_tavily` busca evidencia fáctica en fuentes oficiales
3. `agente_verificador` cruza ambos reportes y emite un veredicto con nivel de confianza

---

<a name="arquitectura-es"></a>
## Arquitectura

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Sub-grafo LangChain                           │
│                                                                       │
│  Chat Input                                                           │
│      │                                                                │
│      ▼                                                                │
│  orquestador_central ◄──── Sonnet 4.5 (LLM)                         │
│      │                ◄──── windows_buffer_memory (10 turnos)        │
│      │ (ai_tool ×3)                                                   │
│      ├──────────────► agente_serp       ◄── GPT-4o Mini + SerpAPI    │
│      ├──────────────► agente_tavily     ◄── GPT-4o     + Tavily      │
│      └──────────────► agente_verificador ◄── GPT-4o                  │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

> **Nota de diagrama:** Las conexiones `ai_tool` son parte del sub-grafo LangChain. El orquestador decide el orden en runtime. La memoria de ventana persiste a lo largo de la sesión de chat.

---

<a name="nodos-es"></a>
## Descripción de nodos

| Nodo | Tipo | Modelo / Tool | Rol |
|------|------|---------------|-----|
| `When chat message received` | Chat Trigger | — | Punto de entrada — recibe la noticia o afirmación a verificar |
| `orquestador_central` | AI Agent | Claude Sonnet 4.5 | Orquestador — gestiona el plan de investigación, sintetiza el informe final |
| `windows_buffer_memory` | Memory Buffer Window | — | Ventana de contexto de 10 turnos para investigaciones multi-turno |
| `agente_serp` | Agent Tool | GPT-4o Mini + SerpAPI | Rastreador de prensa — mapea cobertura mediática actual y tipos de fuente |
| `agente_tavily` | Agent Tool | GPT-4o + Tavily | Investigador de datos — recupera declaraciones oficiales y evidencia fáctica |
| `agente_verificador` | Agent Tool | GPT-4o | Auditor lógico — cruza ambos reportes y emite veredicto con nivel de confianza |
| `Sonnet 4.5` | LM Chat Anthropic | claude-sonnet-4-5 | LLM para el orquestador |
| `GPT 4o Mini` | LM Chat OpenAI | gpt-4o-mini | LLM para el agente SERP |
| `GPT 4o` | LM Chat OpenAI | gpt-4o | LLM compartido para Tavily y verificador |
| `SerpAPI` | Tool SERP | — | Búsqueda en Google — mapea narrativa mediática |
| `Tavily` | Tool Tavily | — | Investigación profunda — recupera fuentes primarias y datos oficiales |

### Resumen de system prompts

**`agente_serp`** (GPT-4o Mini + SerpAPI)
Rastreador de prensa. Identifica las 5 fuentes principales que reportan la noticia y las clasifica en: Medios oficiales / Redes sociales / Blogs de opinión.

**`agente_tavily`** (GPT-4o + Tavily)
Investigador de datos. Busca evidencia fáctica — estadísticas, comunicados oficiales, artículos de investigación. Prioriza precisión técnica sobre narrativa.

**`agente_verificador`** (GPT-4o)
Auditor de lógica independiente. Recibe los reportes de `serp` y `tavily`, señala contradicciones entre narrativa mediática y datos oficiales, y emite un veredicto con nivel de confianza: **Bajo / Medio / Alto**.

### Formato de output del orquestador

El `orquestador_central` entrega al usuario un informe estructurado:
1. La noticia o afirmación evaluada
2. Análisis de fuentes mediáticas (reporte de `agente_serp`)
3. Evidencia fáctica encontrada (reporte de `agente_tavily`)
4. Contradicciones detectadas
5. Veredicto final con nivel de confianza

---

<a name="diseno-es"></a>
## Decisiones de diseño

### Por qué modelos diferentes por agente

La asignación de modelos es deliberada y refleja una estrategia de costo/capacidad:

| Agente | Modelo | Justificación |
|--------|--------|---------------|
| `orquestador_central` | Claude Sonnet 4.5 | Meta-razonamiento, síntesis de múltiples fuentes, redacción del informe final |
| `agente_serp` | GPT-4o Mini | Tarea bien acotada: clasificar URLs y titulares. No requiere razonamiento profundo |
| `agente_tavily` | GPT-4o | Necesita interpretar documentos técnicos, comunicados oficiales, papers |
| `agente_verificador` | GPT-4o | Razonamiento lógico comparativo — detectar contradicciones requiere capacidad analítica |

### Por qué Window Buffer Memory en el orquestador y no en los tools

La memoria persiste en el nivel del orquestador porque él mantiene el estado de la investigación. Los tools son stateless por diseño — ejecutan una tarea atómica y devuelven su resultado. Esto simplifica el comportamiento de cada especialista y hace la investigación acumulable sin complicar los prompts de los tools.

### Por qué SerpAPI para prensa y Tavily para datos

SerpAPI es mejor para mapear el ecosistema mediático actual (qué se dice, dónde, con qué tono). Tavily está optimizado para recuperar contenido de fuentes primarias con mayor fidelidad — ideal para evidencia fáctica. La separación de estas responsabilidades evita que un solo tool mezcle narrativa con dato.

### Por qué el verificador es independiente y no el orquestador

Si el orquestador sintetizara directamente los reportes, introduciría su sesgo de planificación en el veredicto. Delegar la verificación a un agente independiente implementa un **principio de separación de poderes** dentro del sistema — el juez no es el mismo que el investigador.

---

<a name="setup-es"></a>
## Setup y credenciales

| Credencial | Nodo | Notas |
|-----------|------|-------|
| `anthropicApi` | Sonnet 4.5 | API key de Anthropic |
| `openAiApi` | GPT 4o Mini, GPT 4o | API key de OpenAI |
| SerpAPI key | Nodo tool SerpAPI | Registrar en serpapi.com — plan gratuito: 100 búsquedas/mes |
| Tavily API key | Nodo tool Tavily | Registrar en tavily.com — plan gratuito disponible |

🔴 **SerpAPI y Tavily son servicios externos con cuotas.** En uso intensivo o evaluaciones masivas, monitorear el consumo. El flujo no tiene manejo de rate limit integrado.

🟡 **La memoria de ventana es de sesión** — se resetea al iniciar una nueva conversación en el Chat Trigger. No hay persistencia entre sesiones.

---

<a name="uso-es"></a>
## Ejemplo de uso

**Input:**
```
Verifica esta noticia: "México alcanzó el 100% de energía renovable en 2024"
```

**Estructura de output esperada:**
```
📋 NOTICIA EVALUADA:
[Afirmación exacta del usuario]

📰 ANÁLISIS DE FUENTES MEDIÁTICAS:
[Reporte de agente_serp: quién lo dice, tipo de fuente]

🔬 EVIDENCIA FÁCTICA:
[Reporte de agente_tavily: datos oficiales, comunicados, estadísticas]

⚠️ CONTRADICCIONES DETECTADAS:
[Lista de discrepancias entre narrativa y dato]

🏁 VEREDICTO: [Bajo / Medio / Alto confianza]
[Justificación del nivel de confianza]
```

---

<a name="limitaciones-es"></a>
## Limitaciones y problemas conocidos

🔴 **El veredicto es probabilístico, no determinista.** El `agente_verificador` emite un nivel de confianza basado en razonamiento LLM, no en una función de scoring formal. Para aplicaciones de alto riesgo, el output debe ser revisado por un humano.

🔴 **SerpAPI retorna resultados de Google Search — sujeto a SEO y personalización.** Los resultados pueden variar por región o idioma. El `agente_serp` puede recibir cobertura sesgada dependiendo del query.

🟡 **Tavily tiene un límite de profundidad de recuperación.** Para noticias con documentos muy técnicos o paywalls, el agente puede no acceder al contenido completo.

🟡 **El flujo no persiste los informes.** Los reportes de cada sesión se pierden al cerrar el chat. Para auditoría o trazabilidad, considera agregar un nodo de escritura a Google Sheets o Supabase al final del pipeline.

🟡 **Sin idioma forzado en los tools.** Si el input está en español, los tools tienden a responder en español, pero no está garantizado. Para consistencia de idioma, especifica el idioma de salida en cada system prompt.

---

<a name="didacticas-es"></a>
## Notas didácticas

Este flujo ilustra patrones arquitectónicos relevantes para sistemas de verificación y agentes con múltiples fuentes de verdad:

1. **Separación de fuente vs. dato.** `agente_serp` mapea la narrativa (lo que se dice). `agente_tavily` busca evidencia (lo que se puede comprobar). Esta distinción epistemológica es el núcleo del fact-checking y de cualquier sistema RAG con múltiples retrieval sources.

2. **Agente verificador como capa de contraste independiente.** En lugar de que el orquestador sintetice directamente, delegar la verificación a un tercer agente implementa una separación de responsabilidades que reduce el sesgo de confirmación en el sistema.

3. **Modelo sizing por tarea.** GPT-4o Mini para clasificación de titulares, GPT-4o para razonamiento analítico, Claude para síntesis compleja. Este patrón de "right-sizing" es directamente aplicable a pipelines de producción donde el costo por token es una restricción real.

4. **Memoria en el nivel correcto.** La memoria de ventana vive en el orquestador, no en los tools. Esto permite que la investigación sea acumulable sin añadir estado a cada especialista.

---

<a name="estructura-es"></a>
## Estructura de archivos

```
n8n-agent-flows/
└── fact-checker/
    ├── fact-checker.json     # Workflow exportable de n8n
    └── README.md             # Este archivo
```

---

*Parte del repositorio [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) — una colección de workflows de n8n para sistemas AI, pipelines RAG y automatización de procesos.*
