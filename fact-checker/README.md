# Fact Checker — Multi-Agent News Verification System

**Sistema de verificación de noticias basado en contraste de fuentes mediáticas vs. datos fácticos.**  
**News verification system built on media narrative vs. factual data contrast.**

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

Este flujo implementa una **arquitectura de Verificación y Contraste** con un orquestador central (Claude Sonnet 4.5) que coordina tres agentes especialistas para auditar la veracidad de una noticia o afirmación.

El flujo de trabajo es:
1. `agente_serp` rastrea cómo los medios están reportando la noticia
2. `agente_tavily` busca evidencia fáctica en fuentes oficiales
3. `agente_verificador` cruza ambos reportes y emite un veredicto con nivel de confianza

This workflow implements a **Verification-and-Contrast architecture** where a central orchestrator (Claude Sonnet 4.5) coordinates three specialist agents to audit the veracity of a news item or claim.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         LangChain Sub-graph                           │
│                                                                       │
│  Chat Input                                                            │
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

> **Nota de diagrama / Diagram note:** Las conexiones `ai_tool` son parte del sub-grafo LangChain. El orquestador decide el orden en runtime. La memoria de ventana persiste a lo largo de la sesión de chat.  
> `ai_tool` connections belong to n8n's LangChain sub-graph. The orchestrator controls invocation order at runtime. Window memory persists across the chat session.

---

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
Rastreador de prensa. Identifica las 5 fuentes principales que reportan la noticia y las clasifica en: Medios oficiales / Redes sociales / Blogs de opinión.

**`agente_tavily`** (GPT-4o + Tavily)  
Investigador de datos. Busca evidencia fáctica — estadísticas, comunicados oficiales, artículos de investigación. Prioriza precisión técnica sobre narrativa.

**`agente_verificador`** (GPT-4o)  
Auditor de lógica independiente. Recibe los reportes de `serp` y `tavily`, señala contradicciones entre narrativa mediática y datos oficiales, y emite un veredicto con nivel de confianza: **Bajo / Medio / Alto**.

### Orchestrator Output Format

El `orquestador_central` entrega al usuario un informe estructurado:
1. La noticia o afirmación evaluada
2. Análisis de fuentes mediáticas (reporte de `agente_serp`)
3. Evidencia fáctica encontrada (reporte de `agente_tavily`)
4. Contradicciones detectadas
5. Veredicto final con nivel de confianza

---

## Design Decisions

### Por qué modelos diferentes por agente

La asignación de modelos es deliberada y refleja una estrategia de costo/capacidad:

| Agente | Modelo | Justificación |
|--------|--------|---------------|
| `orquestador_central` | Claude Sonnet 4.5 | Meta-razonamiento, síntesis de múltiples fuentes, redacción del informe final |
| `agente_serp` | GPT-4o Mini | Tarea bien acotada: clasificar URLs y titulares. No requiere razonamiento profundo |
| `agente_tavily` | GPT-4o | Necesita interpretar documentos técnicos, comunicados oficiales, papers |
| `agente_verificador` | GPT-4o | Razonamiento lógico comparativo — detectar contradicciones requiere capacidad analítica |

> The model selection isn't default or arbitrary — it reflects a cost-quality trade-off at each layer of the pipeline. This is a design decision worth documenting explicitly.

### Por qué Window Buffer Memory en el orquestador y no en los tools

La memoria persiste en el nivel del orquestador porque él mantiene el estado de la investigación. Los tools son stateless por diseño — ejecutan una tarea atómica y devuelven su resultado. Esto simplifica el comportamiento de cada especialista y hace la investigación acumulable sin complicar los prompts de los tools.

### Por qué SerpAPI para prensa y Tavily para datos

SerpAPI es mejor para mapear el ecosistema mediático actual (qué se dice, dónde, con qué tono). Tavily está optimizado para recuperar contenido de fuentes primarias con mayor fidelidad — ideal para evidencia fáctica. La separación de estas responsabilidades evita que un solo tool mezcle narrativa con dato.

### Por qué el verificador es independiente y no el orquestador

Si el orquestador sintetizara directamente los reportes, introduciría su sesgo de planificación en el veredicto. Delegar la verificación a un agente independiente implementa un **principio de separación de poderes** dentro del sistema — el juez no es el mismo que el investigador.

---

## Setup & Credentials

| Credential | Node | Notes |
|-----------|------|-------|
| `anthropicApi` | Sonnet 4.5 | API key de Anthropic |
| `openAiApi` | GPT 4o Mini, GPT 4o | API key de OpenAI |
| SerpAPI key | SerpAPI tool node | Registrar en serpapi.com — plan gratuito: 100 búsquedas/mes |
| Tavily API key | Tavily tool node | Registrar en tavily.com — plan gratuito disponible |

🔴 **SerpAPI y Tavily son servicios externos con cuotas.** En uso intensivo o evaluaciones masivas, monitorear el consumo. El flujo no tiene manejo de rate limit integrado.

🟡 **La memoria de ventana es de sesión** — se resetea al iniciar una nueva conversación en el Chat Trigger. No hay persistencia entre sesiones.

---

## Usage Example

**Input:**
```
Verifica esta noticia: "México alcanzó el 100% de energía renovable en 2024"
```

**Expected output structure:**
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

## Limitations & Known Issues

🔴 **El veredicto es probabilístico, no determinista.** El `agente_verificador` emite un nivel de confianza basado en razonamiento LLM, no en una función de scoring formal. Para aplicaciones de alto riesgo, el output debe ser revisado por un humano.

🔴 **SerpAPI retorna resultados de Google Search — sujeto a SEO y personalización.** Los resultados pueden variar por región o idioma. El `agente_serp` puede recibir cobertura sesgada dependiendo del query.

🟡 **Tavily tiene un límite de profundidad de recuperación.** Para noticias con documentos muy técnicos o paywalls, el agente puede no acceder al contenido completo.

🟡 **El flujo no persiste los informes.** Los reportes de cada sesión se pierden al cerrar el chat. Para auditoría o trazabilidad, considera agregar un nodo de escritura a Google Sheets o Supabase al final del pipeline.

🟡 **Sin idioma forzado en los tools.** Si el input está en español, los tools tienden a responder en español, pero no está garantizado. Para consistencia de idioma, especifica el idioma de salida en cada system prompt.

---

## Didactic Notes

Este flujo ilustra patrones arquitectónicos relevantes para sistemas de verificación y agentes con múltiples fuentes de verdad:

1. **Separación de fuente vs. dato.** `agente_serp` mapea la narrativa (lo que se dice). `agente_tavily` busca evidencia (lo que se puede comprobar). Esta distinción epistemológica es el núcleo del fact-checking y de cualquier sistema RAG con múltiples retrieval sources.

2. **Agente verificador como capa de contraste independiente.** En lugar de que el orquestador sintetice directamente, delegar la verificación a un tercer agente implementa una separación de responsabilidades que reduce el sesgo de confirmación en el sistema.

3. **Modelo sizing por tarea.** GPT-4o Mini para clasificación de titulares, GPT-4o para razonamiento analítico, Claude para síntesis compleja. Este patrón de "right-sizing" es directamente aplicable a pipelines de producción donde el costo por token es una restricción real.

4. **Memoria en el nivel correcto.** La memoria de ventana vive en el orquestador, no en los tools. Esto permite que la investigación sea acumulable sin añadir estado a cada especialista.