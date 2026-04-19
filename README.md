# n8n-agent-flows

> A curated collection of production-grade n8n workflows for AI systems, RAG pipelines, conversational agents, and process automation.

**by [Edgar Darío Arteaga Gaytán](https://github.com/DarioArteaga) · AI & LLM Engineer · Zacatecas, México**

---

**Language / Idioma:** [English](#english-index) · [Español](#español-index)

---

<a name="english-index"></a>
# 🇺🇸 English

## What is this?

This repository contains exportable n8n workflows built and used in real AI projects. Each workflow is documented with:

- **Purpose** — what problem it solves and where it fits in a larger system
- **Architecture** — node breakdown, connection types, and design decisions
- **Setup instructions** — required credentials, environment configuration, and external dependencies
- **Trade-offs and warnings** — known limitations, edge cases, and production considerations

These are not templates or boilerplate. They are concrete implementations with documented reasoning — built to be understood, adapted, and extended.

---

## Repository Structure

```
n8n-agent-flows/
├── README.md
├── rag/
├── invoice-uploader/
├── agente-calculadora/
├── agente-investigador/
├── content-creator/
└── fact-checker/
```

Each folder contains the workflow `.json` file and a `README.md`.

---

## Available Workflows

### RAG

| Workflow | Trigger | Description | Stack |
|---|---|---|---|
| [`rag`](./rag/README.md) | Manual | Ingestion pipeline: Google Drive PDF → recursive chunking → `text-embedding-3-small` → Supabase pgvector insert | n8n · OpenAI · Supabase · LangChain |

### Process Automation

| Workflow | Trigger | Description | Stack |
|---|---|---|---|
| [`invoice-uploader`](./invoice-uploader/README.md) | Daily 00:00 | Gmail fiscal email poller — OR-regex filter (factura/CFDI/comprobante) → attachment fan-out (JS Code node) → Drive upload → mark as read | n8n · Gmail · Google Drive |

### Agents

| Workflow | Trigger | Description | Stack |
|---|---|---|---|
| [`agente-calculadora`](./agente-calculadora/README.md) | Chat | Conversational agent with Calculator tool. Demonstrates the ReAct reasoning loop — stateless by design | n8n · OpenAI · LangChain |
| [`agente-investigador`](./agente-investigador/README.md) | Chat | Research agent with Wikipedia retrieval and session memory window. Demonstrates tool use + short-term context persistence | n8n · OpenAI · Wikipedia · LangChain |

### Content Automation (Multi-Agent)

| Workflow | Trigger | Description | Stack |
|---|---|---|---|
| [`content-creator`](./content-creator/README.md) | Chat | Supervisor–Specialist architecture: Analyst → Writer → Editor pipeline producing Twitter/X threads ≤ 280 chars per block | Claude Sonnet 4.5 · GPT-3.5 Turbo · n8n |
| [`fact-checker`](./fact-checker/README.md) | Chat | Verification-and-Contrast architecture: media narrative (SerpAPI) vs. factual evidence (Tavily) → logic audit → confidence verdict | Claude Sonnet 4.5 · GPT-4o · GPT-4o Mini · SerpAPI · Tavily · n8n |

---

## How to Use a Workflow

1. Download the `.json` file from the workflow folder
2. In your n8n instance: **Import** → **From File** → select the `.json`
3. Connect your own credentials (Google Drive, OpenAI, Supabase, etc.)
4. Read the workflow's `README.md` before running — each one documents required setup, warnings, and design decisions

---

## Documentation Standards

Every workflow README follows this structure:

- **Overview** — what the flow does and why it exists
- **Architecture diagram** — ASCII, distinguishing LangChain sub-graph connections from standard n8n main flow
- **Node breakdown table** — type, model/tool, and role per node
- **Design Decisions** — the *why* behind architectural choices, not just the *what*
- **Setup & Credentials** — required keys and services
- **Limitations & Known Issues** — severity-coded (🔴 critical / 🟡 advisory)
- **Didactic Notes** — patterns worth studying for engineers and students

Language: bilingual EN/ES where context benefits from both.

---

## About

Built by an AI & LLM Engineer with 5+ years shipping end-to-end AI systems: RAG pipelines, conversational agents, LLMOps, and voice interfaces. This repository makes the orchestration layer — the glue between models, data, and APIs — visible and reusable.

These workflows are also used as teaching material in a professional certification program (diplomado) on AI and automation.

[LinkedIn](https://linkedin.com/in/darioarteaga) · [GitHub](https://github.com/DarioArteaga)

---
---

<a name="español-index"></a>
# 🇲🇽 Español

## ¿Qué es esto?

Este repositorio contiene workflows de n8n exportables, construidos y utilizados en proyectos reales de AI. Cada workflow está documentado con:

- **Propósito** — qué problema resuelve y cómo encaja en un sistema más grande
- **Arquitectura** — descripción de nodos, tipos de conexión y decisiones de diseño
- **Instrucciones de configuración** — credenciales requeridas, configuración del entorno y dependencias externas
- **Trade-offs y advertencias** — limitaciones conocidas, casos borde y consideraciones para producción

No son templates ni boilerplate. Son implementaciones concretas con razonamiento documentado — construidas para ser entendidas, adaptadas y extendidas.

---

## Estructura del repositorio

```
n8n-agent-flows/
├── README.md
├── rag/
├── invoice-uploader/
├── agente-calculadora/
├── agente-investigador/
├── content-creator/
└── fact-checker/
```

Cada carpeta contiene el archivo `.json` del workflow y un `README.md`.

---

## Workflows disponibles

### RAG

| Workflow | Trigger | Descripción | Stack |
|---|---|---|---|
| [`rag`](./rag/README.md) | Manual | Pipeline de ingesta: PDF desde Google Drive → chunking recursivo → `text-embedding-3-small` → inserción en Supabase pgvector | n8n · OpenAI · Supabase · LangChain |

### Automatización de procesos

| Workflow | Trigger | Descripción | Stack |
|---|---|---|---|
| [`invoice-uploader`](./invoice-uploader/README.md) | Diario 00:00 | Poller de correos fiscales en Gmail — filtro OR-regex (factura/CFDI/comprobante) → fan-out de adjuntos (nodo Code JS) → subida a Drive → marcar como leído | n8n · Gmail · Google Drive |

### Agentes

| Workflow | Trigger | Descripción | Stack |
|---|---|---|---|
| [`agente-calculadora`](./agente-calculadora/README.md) | Chat | Agente conversacional con herramienta Calculator. Demuestra el loop de razonamiento ReAct — sin estado por diseño | n8n · OpenAI · LangChain |
| [`agente-investigador`](./agente-investigador/README.md) | Chat | Agente investigador con recuperación en Wikipedia y ventana de memoria de sesión. Demuestra uso de herramientas + persistencia de contexto de corto plazo | n8n · OpenAI · Wikipedia · LangChain |

### Automatización de contenido (Multi-Agente)

| Workflow | Trigger | Descripción | Stack |
|---|---|---|---|
| [`content-creator`](./content-creator/README.md) | Chat | Arquitectura Supervisor–Especialistas: pipeline Analista → Redactor → Editor que produce hilos de Twitter/X con bloques ≤ 280 caracteres | Claude Sonnet 4.5 · GPT-3.5 Turbo · n8n |
| [`fact-checker`](./fact-checker/README.md) | Chat | Arquitectura de Verificación y Contraste: narrativa mediática (SerpAPI) vs. evidencia fáctica (Tavily) → auditoría lógica → veredicto con nivel de confianza | Claude Sonnet 4.5 · GPT-4o · GPT-4o Mini · SerpAPI · Tavily · n8n |

---

## Cómo usar un workflow

1. Descarga el archivo `.json` desde la carpeta del workflow
2. En tu instancia de n8n: **Import** → **From File** → selecciona el `.json`
3. Conecta tus propias credenciales (Google Drive, OpenAI, Supabase, etc.)
4. Lee el `README.md` del workflow antes de ejecutarlo — cada uno documenta la configuración requerida, advertencias y decisiones de diseño

---

## Estándares de documentación

Cada README de workflow sigue esta estructura:

- **Overview** — qué hace el flujo y por qué existe
- **Diagrama de arquitectura** — ASCII, distinguiendo conexiones del sub-grafo LangChain del flujo main estándar de n8n
- **Tabla de nodos** — tipo, modelo/herramienta y rol por nodo
- **Decisiones de diseño** — el *por qué* detrás de las decisiones arquitectónicas, no solo el *qué*
- **Setup y credenciales** — keys y servicios requeridos
- **Limitaciones y problemas conocidos** — con severidad codificada (🔴 crítico / 🟡 informativo)
- **Notas didácticas** — patrones relevantes para engineers y estudiantes

Idioma: bilingüe EN/ES donde el contexto se beneficia de ambos.

---

## Sobre el autor

Construido por un AI & LLM Engineer con 5+ años entregando sistemas AI end-to-end: pipelines RAG, agentes conversacionales, LLMOps e interfaces de voz. Este repositorio hace visible y reutilizable la capa de orquestación — el tejido conectivo entre modelos, datos y APIs.

Estos workflows también se usan como material didáctico en un diplomado de AI y automatización.

[LinkedIn](https://linkedin.com/in/darioarteaga) · [GitHub](https://github.com/DarioArteaga)