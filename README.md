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
├── README.md                        ← You are here
│
├── rag/                             ← RAG ingestion and retrieval pipelines
│   └── ingest_RAG/
│       ├── ingest_RAG.json
│       └── README.md
│
├── process-automation/              ← Document processing and task automation
│   └── invoice-uploader/
│       ├── invoice-uploader.json
│       └── README.md
│
├── agents/                          ← Conversational and process agents
│   ├── agente-calculadora/
│   │   ├── agente-calculadora.json
│   │   └── README.md
│   └── agente-investigador/
│       ├── agente-investigador.json
│       └── README.md
│
└── voice/                           ← STT / TTS integrations (coming soon)
```

---

## Available Workflows

### RAG

| Workflow | Description | Stack |
|---|---|---|
| [`ingest_RAG`](./rag/ingest_RAG/README.md) | Manually triggered ingestion pipeline. Downloads a document from Google Drive, chunks it with a Recursive Character Text Splitter, generates embeddings with `text-embedding-3-small`, and inserts vectors into a Supabase pgvector table. | n8n · OpenAI · Supabase · LangChain |

### Process Automation

| Workflow | Description | Stack |
|---|---|---|
| [`invoice-uploader`](./process-automation/invoice-uploader/README.md) | Polls Gmail daily for unread emails with attachments, filters by fiscal keywords (factura, CFDI, comprobante), fans out each attachment into an individual item, and uploads them to a Google Drive folder. Marks the email as read on success. | n8n · Gmail · Google Drive |

### Agents

| Workflow | Description | Stack |
|---|---|---|
| [`agente-calculadora`](./agents/agente-calculadora/README.md) | Conversational agent exposed through n8n's built-in chat UI. Uses a Calculator tool for arithmetic and demonstrates the ReAct reasoning loop — the model decides on each turn whether to invoke the tool or answer directly. Stateless by design. | n8n · OpenAI · LangChain |
| [`agente-investigador`](./agents/agente-investigador/README.md) | Conversational research agent with Wikipedia retrieval and a session memory window. Demonstrates tool use combined with short-term context persistence across turns. Each browser session maintains its own isolated memory. | n8n · OpenAI · Wikipedia · LangChain |

---

## How to Use a Workflow

1. Download the `.json` file from the workflow folder
2. In your n8n instance: **Import** → **From File** → select the `.json`
3. Connect your own credentials (Google Drive, OpenAI, Supabase, etc.)
4. Read the workflow's `README.md` before running — each one documents required setup steps, warnings, and configuration details

---

## About

Built by an AI & LLM Engineer with 5+ years shipping end-to-end AI systems: RAG pipelines, conversational agents, LLMOps, and voice interfaces. This repository makes the orchestration layer — the glue between models, data, and APIs — visible and reusable.

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
├── README.md                        ← Estás aquí
│
├── rag/                             ← Pipelines de ingesta y recuperación RAG
│   └── ingest_RAG/
│       ├── ingest_RAG.json
│       └── README.md
│
├── process-automation/              ← Procesamiento de documentos y automatización de tareas
│   └── invoice-uploader/
│       ├── invoice-uploader.json
│       └── README.md
│
├── agents/                          ← Agentes conversacionales y de procesos
│   ├── agente-calculadora/
│   │   ├── agente-calculadora.json
│   │   └── README.md
│   └── agente-investigador/
│       ├── agente-investigador.json
│       └── README.md
│
└── voice/                           ← Integraciones STT / TTS (próximamente)
```

---

## Workflows disponibles

### RAG

| Workflow | Descripción | Stack |
|---|---|---|
| [`ingest_RAG`](./rag/ingest_RAG/README.md) | Pipeline de ingesta con activación manual. Descarga un documento desde Google Drive, lo fragmenta con un Recursive Character Text Splitter, genera embeddings con `text-embedding-3-small` e inserta los vectores en una tabla pgvector de Supabase. | n8n · OpenAI · Supabase · LangChain |

### Automatización de procesos

| Workflow | Descripción | Stack |
|---|---|---|
| [`invoice-uploader`](./process-automation/invoice-uploader/README.md) | Consulta Gmail diariamente en busca de correos no leídos con adjuntos, filtra por palabras clave fiscales (factura, CFDI, comprobante), separa cada adjunto en un ítem individual y los sube a una carpeta de Google Drive. Marca el correo como leído al completar. | n8n · Gmail · Google Drive |

### Agentes

| Workflow | Descripción | Stack |
|---|---|---|
| [`agente-calculadora`](./agents/agente-calculadora/README.md) | Agente conversacional expuesto a través de la UI de chat integrada en n8n. Usa una herramienta Calculator para aritmética y demuestra el loop de razonamiento ReAct — el modelo decide en cada turno si invocar la herramienta o responder directamente. Sin estado por diseño. | n8n · OpenAI · LangChain |
| [`agente-investigador`](./agents/agente-investigador/README.md) | Agente investigador conversacional con recuperación en Wikipedia y una ventana de memoria de sesión. Demuestra el uso de herramientas combinado con persistencia de contexto de corto plazo entre turnos. Cada sesión del navegador mantiene su propia memoria aislada. | n8n · OpenAI · Wikipedia · LangChain |

---

## Cómo usar un workflow

1. Descarga el archivo `.json` desde la carpeta del workflow
2. En tu instancia de n8n: **Import** → **From File** → selecciona el `.json`
3. Conecta tus propias credenciales (Google Drive, OpenAI, Supabase, etc.)
4. Lee el `README.md` del workflow antes de ejecutarlo — cada uno documenta los pasos de configuración requeridos, advertencias y detalles de configuración

---

## Sobre el autor

Construido por un AI & LLM Engineer con 5+ años entregando sistemas AI end-to-end: pipelines RAG, agentes conversacionales, LLMOps e interfaces de voz. Este repositorio hace visible y reutilizable la capa de orquestación — el tejido conectivo entre modelos, datos y APIs.

[LinkedIn](https://linkedin.com/in/darioarteaga) · [GitHub](https://github.com/DarioArteaga)