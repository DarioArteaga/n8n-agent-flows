# invoice-uploader

> **n8n workflow** — Automated invoice collector: polls Gmail daily at midnight for unread emails with attachments, filters by fiscal keywords, separates each attachment into an individual item, and uploads them to a Google Drive folder. Marks the email as read on success.

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

This workflow automates the collection of fiscal documents (invoices, receipts, CFDI) that arrive as email attachments. It runs continuously in the background, inspecting every new Gmail message with an attachment and routing it based on whether it looks like an invoice.

The pipeline handles the full flow from detection to storage:

```
[Gmail Trigger] → [Invoice Filter] → [Split Attachments] → [Upload to Drive] → [Mark as Read]
                                  ↓
                            [Non-invoice: NoOp]
```

> **Note on the Drive folder:** The workflow comes configured with a placeholder Google Drive folder ID. You will need to connect your own Google Drive account and set your own destination folder — see [Prerequisites](#prerequisites-en) for details.

---

<a name="architecture-en"></a>
## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          invoice-uploader                                 │
│                                                                            │
│  ┌─────────────────┐    ┌────────────┐                                    │
│  │ Gmail Nuevos    │───▶│ Es Factura │                                    │
│  │ Correos         │    │ (IF node)  │                                    │
│  │ (poll/daily 00:00)    │    └─────┬──────┘                                    │
│  └─────────────────┘          │                                           │
│                         true  │  false                                    │
│                    ┌──────────┘  └──────────────┐                        │
│                    ▼                             ▼                        │
│          ┌──────────────────┐        ┌────────────────────┐              │
│          │ Separar Adjuntos │        │ No Operation       │              │
│          │  (Code node)     │        │ (silent discard)   │              │
│          └────────┬─────────┘        └────────────────────┘              │
│                   │  (one item per attachment)                            │
│                   ▼                                                       │
│          ┌─────────────────────┐                                         │
│          │ Subir a Google Drive│                                         │
│          └────────┬────────────┘                                         │
│                   ▼                                                       │
│          ┌─────────────────────┐                                         │
│          │ Mark a message      │                                         │
│          │ as read             │                                         │
│          └─────────────────────┘                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

---

<a name="nodes-en"></a>
## Node-by-Node Breakdown

### 1. `Gmail Nuevos Correos`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.gmailTrigger` |
| Poll frequency | Daily at 00:00 |
| Filter | Unread emails with attachments (`has:attachment`) |
| Output format | Full message data + binary attachments |
| Credentials | `gmailOAuth2` |

Polls Gmail once a day, at midnight, for unread messages that contain at least one attachment. The `simple: false` setting returns the full Gmail message object — including headers, body, and binary attachment data — rather than a simplified representation.

> **Note:** Daily polling at midnight means up to a 24-hour delay between email arrival and processing. Suitable for non-time-sensitive invoice collection.

---

### 2. `Es Factura`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.if` |
| Logic | OR — subject OR body must match |
| Subject regex | `(factura\|invoice\|comprobante\|cfdi\|recibo)` |
| Body regex | `(factura\|comprobante fiscal\|cfdi)` |
| Case sensitivity | Applied via `.toLowerCase()` |

Routes emails based on whether they look like fiscal documents. The filter uses two independent regex checks combined with `OR`:
- **Condition A:** Subject contains any of: `factura`, `invoice`, `comprobante`, `cfdi`, `recibo`
- **Condition B:** Body (plain text or HTML) contains any of: `factura`, `comprobante fiscal`, `cfdi`

A match on **either** condition routes the email to the processing branch. Non-matching emails go to the `NoOp` node and are silently discarded — **they are not marked as read**, so they remain in the unread pool.

> ⚠️ The OR combinator means any email mentioning "factura" in its body — even if unrelated — will be processed. See [Limitations](#limitations-en).

---

### 3. `Separar Adjuntos`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.code` |
| Language | JavaScript |
| Input | Single email item with multiple `attachment_N` binary keys |
| Output | One item per attachment, each carrying email metadata |

This is the most important structural node in the workflow. A single email can have multiple attachments, but n8n's default item model would pass them all together. This Code node fans them out:

```javascript
// For each email item, extract each attachment_N binary key
// and emit it as a standalone item carrying:
// - emailSubject, emailFrom, emailDate (from email metadata)
// - fileName (from binary data)
// - the binary attachment itself
```

The result is that downstream nodes (`Subir a Google Drive`, `Mark as read`) receive one item per attachment, not one item per email. This enables parallel upload of multiple files from the same email.

---

### 4. `Subir a Google Drive`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.googleDrive` |
| Operation | Upload file |
| File name | `={{ $json.fileName }}` (from attachment metadata) |
| Destination | `Facturas` folder (user-configured) |
| Credentials | `googleDriveOAuth2Api` |

Uploads each attachment to the configured Drive folder, using the original filename as-is. **You must connect your own Google Drive account and configure your own destination folder** — the workflow does not include a shared Drive location.

> ⚠️ There is no file type filter. All attachments — PDF, XML, PNG, DOCX, ZIP — are uploaded regardless of format. See [Limitations](#limitations-en).

---

### 5. `Mark a message as read`
| Property | Value |
|---|---|
| Type | `n8n-nodes-base.gmail` |
| Operation | `markAsRead` |
| Message ID | `={{ $('Gmail Nuevos Correos').item.json.id }}` |
| Credentials | `gmailOAuth2` |

Marks the original email as read after a successful upload. This serves as the workflow's implicit success signal: only emails whose attachments were fully uploaded get marked as read.

> ⚠️ Because `Separar Adjuntos` fans out one item per attachment, this node runs inside a multi-item context. The Message ID expression **must use `.first()`**, not `.item`:
> ```
> {{ $('Gmail Nuevos Correos').first().json.id }}   ✓
> ```
> Using `.item` will cause n8n to throw *"Can't determine which item to use"* when more than one attachment is present. See [Limitations](#limitations-en) for details.

---

### 6. `No Operation, do nothing`
The `false` branch of `Es Factura`. Non-invoice emails are silently discarded — no action, no notification, no logging. They remain **unread** in Gmail.

---

<a name="prerequisites-en"></a>
## Prerequisites & Credentials

### Required Credentials in n8n

| Credential Name | Type | Used By | How to Set Up |
|---|---|---|---|
| Gmail (your account) | `gmailOAuth2` | `Gmail Nuevos Correos`, `Mark a message as read` | [n8n Gmail Docs](https://docs.n8n.io/integrations/builtin/credentials/google/) |
| Google Drive (your account) | `googleDriveOAuth2Api` | `Subir a Google Drive` | Same Google OAuth2 credentials |

### Configuring the Destination Folder

1. Open the `Subir a Google Drive` node
2. Connect your Google Drive account via OAuth2
3. In the **Folder** field, use the folder picker to select or create your destination folder
4. The workflow will upload all matched attachments to that folder

### Gmail Permissions Required

The Gmail OAuth2 credential needs the following scopes:
- `https://www.googleapis.com/auth/gmail.readonly` — to read messages and attachments
- `https://www.googleapis.com/auth/gmail.modify` — to mark messages as read

---

<a name="specs-en"></a>
## Technical Specs

| Component | Technology |
|---|---|
| Workflow engine | n8n (`executionOrder: v1`) |
| Email source | Gmail (OAuth2, poll trigger) |
| Poll frequency | Daily at 00:00 |
| Invoice detection | Regex on subject + body (OR logic) |
| Attachment handling | JavaScript fan-out (one item per attachment) |
| File storage | Google Drive (user-configured folder) |
| Workflow state | **Active** — runs continuously |

---

<a name="design-en"></a>
## Design Decisions

### Why polling instead of a push trigger?
Gmail does not natively support webhooks in n8n without additional setup (Google Pub/Sub). Polling once a day at midnight is a deliberate trade-off: it eliminates continuous API usage at the cost of up-to-24-hour latency between email arrival and file upload.

### Why OR logic on the invoice filter?
The filter uses OR (subject OR body) to maximize recall — missing a real invoice is worse than processing a false positive. The trade-off is that promotional emails or newsletters mentioning "factura" in their body may slip through. Adjust to AND logic if false positives become a problem.

### Why a Code node for attachment splitting?
n8n's Gmail trigger returns all attachments under a single item as `attachment_0`, `attachment_1`, etc. There is no built-in node that fans out binary data from a single item into multiple items. The Code node is the standard solution for this pattern.

### Why mark as read at the end?
Using "mark as read" as the final step creates an implicit idempotency mechanism: if the workflow fails mid-execution (e.g., Drive upload error), the email remains unread and will be picked up on the next poll cycle. This is not a guarantee — it is a best-effort safeguard.

---

<a name="limitations-en"></a>
## Limitations & Warnings

### 🔴 No File Type Filtering
All attachments are uploaded regardless of format. This means images, ZIP files, Word documents, and Excel sheets from invoice emails will be stored alongside the actual fiscal documents.

**Impact:** The `Facturas` folder accumulates noise — company logos, signatures, and unrelated files embedded in invoice emails.

**Mitigation:** Add a filter after `Separar Adjuntos` that checks `$json.fileName` against allowed extensions:
```
{{ ['pdf', 'xml', 'zip'].includes($json.fileName.split('.').pop().toLowerCase()) }}
```

For Mexican fiscal context (CFDI), the relevant formats are `.xml` (the CFDI itself) and `.pdf` (the human-readable representation).

---

### 🔴 Redundant `markAsRead` Calls + Item Resolution Error
Because `Separar Adjuntos` produces one item per attachment, the `Mark a message as read` node runs inside a multi-item context. If the Message ID expression uses `.item`:

```
{{ $('Gmail Nuevos Correos').item.json.id }}   ← will fail
```

n8n cannot determine which item to pair and throws:
> *"Can't determine which item to use"*

**Fix:** Replace `.item` with `.first()` to resolve the original email unambiguously:

```
{{ $('Gmail Nuevos Correos').first().json.id }}   ✓
```

`.first()` always returns the source email regardless of how many attachment items are in flight downstream. This also eliminates the redundant calls: since the same ID is marked read N times (once per attachment), using `.first()` is both the correct and the safe expression.

---

### 🟡 Up to 24-Hour Processing Latency
The daily trigger fires once at midnight. An invoice that arrives at 08:00 will not be uploaded until the following midnight run — up to 16 hours later.

**Impact:** Acceptable for batch accounting workflows and end-of-day reconciliation. Not acceptable if the Drive folder feeds a downstream process that needs invoices available within minutes of receipt.

**Mitigation:** If lower latency is needed, increase the frequency to hourly or every few hours without significantly impacting Gmail API quota.

---

### 🟡 False Positives on Invoice Filter
The OR logic means any email containing "factura" or "cfdi" anywhere in the body — including marketing emails, bank notifications, or newsletters — will be routed to the upload branch.

**Mitigation:** Switch the combinator to AND (subject AND body must match), or add a sender allowlist using the Gmail query field (`from:billing@provider.com`).

---

### 🟡 Non-invoice Emails Are Never Marked as Read
Emails that fail the `Es Factura` filter go to `NoOp` and remain unread. They will be re-evaluated on every subsequent poll.

**Impact:** If Gmail has a large backlog of unread emails with attachments that are not invoices, the workflow will repeatedly process and discard them, consuming API quota unnecessarily.

**Mitigation:** Mark all processed emails as read regardless of the filter result — or use a Gmail label instead of read/unread status as the processing flag.

---

### 🟡 Hardcoded Destination Folder
The Drive folder ID is embedded in the node configuration. There is no parameterization.

**Mitigation:** Use the Drive folder picker to select your own folder when setting up the workflow.

---

### 🟡 No Error Handling
If the Drive upload fails (quota exceeded, permission error, network timeout), the workflow stops silently. The email remains unread, so it will be retried on the next poll — but there is no notification, log, or dead-letter queue.

---

<a name="usecases-en"></a>
## Recommended Use Cases

**Well-suited for:**
- ✅ Personal or small-business invoice collection (low email volume)
- ✅ Centralizing fiscal documents from multiple senders into a single Drive folder
- ✅ Reducing manual download-and-upload work for accounting workflows
- ✅ First step in a larger pipeline (e.g., Drive → accounting software)

**Not suited for:**
- ❌ High-volume inboxes where non-invoice emails with attachments are frequent
- ❌ Environments where file type control is critical (without the extension filter)
- ❌ Multi-account setups (one workflow instance per Gmail account)
- ❌ Production environments requiring audit trails or error notifications

---

<a name="extensions-en"></a>
## Extension Ideas

| Enhancement | Approach |
|---|---|
| File type filter | Add an `IF` node after `Separar Adjuntos` to allow only `.pdf` and `.xml` |
| Sender allowlist | Restrict the Gmail trigger query to known invoice senders (`from:...`) |
| Folder routing by sender | Use a `Switch` node to route to different Drive folders per sender domain |
| CFDI XML parsing | Add a Code node to extract RFC, amount, and UUID from the XML before uploading |
| Spreadsheet log | Append email metadata and filename to a Google Sheet on each successful upload |
| Error notifications | Add an error branch → Slack or email alert on Drive upload failure |
| Reduce redundant `markAsRead` | Collect email IDs after split, deduplicate, and mark once per email |
| Label instead of read status | Use Gmail labels as processing flags for more robust idempotency |

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

Este workflow automatiza la recolección de documentos fiscales (facturas, comprobantes, CFDI) que llegan como adjuntos de correo electrónico. Corre continuamente en segundo plano, revisando cada nuevo mensaje de Gmail con adjunto y enrutándolo según si parece ser una factura.

El pipeline maneja el flujo completo desde la detección hasta el almacenamiento:

```
[Gmail Trigger] → [Filtro Factura] → [Separar Adjuntos] → [Subir a Drive] → [Marcar como leído]
                                  ↓
                        [No factura: NoOp]
```

> **Nota sobre la carpeta de Drive:** El workflow viene configurado con un ID de carpeta de Google Drive de ejemplo. Deberás conectar tu propia cuenta de Google Drive y configurar tu propia carpeta destino — consulta [Requisitos](#requisitos-es) para más detalles.

---

<a name="arquitectura-es"></a>
## Arquitectura

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          invoice-uploader                                 │
│                                                                            │
│  ┌─────────────────┐    ┌────────────┐                                    │
│  │ Gmail Nuevos    │───▶│ Es Factura │                                    │
│  │ Correos         │    │ (nodo IF)  │                                    │
│  │ (poll/daily 00:00)    │    └─────┬──────┘                                    │
│  └─────────────────┘          │                                           │
│                         true  │  false                                    │
│                    ┌──────────┘  └──────────────┐                        │
│                    ▼                             ▼                        │
│          ┌──────────────────┐        ┌────────────────────┐              │
│          │ Separar Adjuntos │        │ No Operation       │              │
│          │  (nodo Code)     │        │ (descarte silente) │              │
│          └────────┬─────────┘        └────────────────────┘              │
│                   │  (un ítem por adjunto)                                │
│                   ▼                                                       │
│          ┌─────────────────────┐                                         │
│          │ Subir a Google Drive│                                         │
│          └────────┬────────────┘                                         │
│                   ▼                                                       │
│          ┌─────────────────────┐                                         │
│          │ Mark a message      │                                         │
│          │ as read             │                                         │
│          └─────────────────────┘                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

---

<a name="nodos-es"></a>
## Descripción de nodos

### 1. `Gmail Nuevos Correos`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.gmailTrigger` |
| Frecuencia de polling | Diario a las 00:00 |
| Filtro | Correos no leídos con adjuntos (`has:attachment`) |
| Formato de salida | Datos completos del mensaje + adjuntos binarios |
| Credenciales | `gmailOAuth2` |

Consulta Gmail una vez al día, a las 00:00, en busca de mensajes no leídos que contengan al menos un adjunto. La configuración `simple: false` devuelve el objeto completo del mensaje de Gmail — incluyendo encabezados, cuerpo y datos binarios de los adjuntos — en lugar de una representación simplificada.

> **Nota:** El polling diario a medianoche implica hasta 24 horas de latencia entre la llegada del correo y su procesamiento. Adecuado para recolección de facturas que no requiere tiempo real.

---

### 2. `Es Factura`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.if` |
| Lógica | OR — asunto O cuerpo deben coincidir |
| Regex asunto | `(factura\|invoice\|comprobante\|cfdi\|recibo)` |
| Regex cuerpo | `(factura\|comprobante fiscal\|cfdi)` |
| Sensibilidad a mayúsculas | Aplicada mediante `.toLowerCase()` |

Enruta correos según si parecen documentos fiscales. El filtro usa dos verificaciones regex independientes combinadas con `OR`:
- **Condición A:** El asunto contiene alguno de: `factura`, `invoice`, `comprobante`, `cfdi`, `recibo`
- **Condición B:** El cuerpo (texto plano o HTML) contiene alguno de: `factura`, `comprobante fiscal`, `cfdi`

Una coincidencia en **cualquiera** de las condiciones enruta el correo a la rama de procesamiento. Los correos que no coinciden van al nodo `NoOp` y se descartan silenciosamente — **no se marcan como leídos**, por lo que permanecen en el pool de no leídos.

> ⚠️ El combinador OR significa que cualquier correo que mencione "factura" en su cuerpo — aunque sea irrelevante — será procesado. Consulta [Limitaciones](#limitaciones-es).

---

### 3. `Separar Adjuntos`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| Lenguaje | JavaScript |
| Entrada | Ítem único de correo con múltiples claves binarias `attachment_N` |
| Salida | Un ítem por adjunto, cada uno con los metadatos del correo |

Este es el nodo estructural más importante del workflow. Un solo correo puede tener múltiples adjuntos, pero el modelo de ítems por defecto de n8n los pasaría todos juntos. Este nodo Code los separa en fan-out:

```javascript
// Por cada ítem de correo, extrae cada clave binaria attachment_N
// y emite un ítem independiente que lleva:
// - emailSubject, emailFrom, emailDate (de los metadatos del correo)
// - fileName (de los datos binarios)
// - el adjunto binario en sí
```

El resultado es que los nodos aguas abajo (`Subir a Google Drive`, `Mark as read`) reciben un ítem por adjunto, no uno por correo. Esto permite la carga paralela de múltiples archivos del mismo correo.

---

### 4. `Subir a Google Drive`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.googleDrive` |
| Operación | Subir archivo |
| Nombre del archivo | `={{ $json.fileName }}` (de los metadatos del adjunto) |
| Destino | Carpeta `Facturas` (configurable por el usuario) |
| Credenciales | `googleDriveOAuth2Api` |

Sube cada adjunto a la carpeta de Drive configurada, usando el nombre de archivo original tal cual. **Debes conectar tu propia cuenta de Google Drive y configurar tu propia carpeta destino** — el workflow no incluye una ubicación de Drive compartida.

> ⚠️ No hay filtro de tipo de archivo. Todos los adjuntos — PDF, XML, PNG, DOCX, ZIP — se suben sin importar el formato. Consulta [Limitaciones](#limitaciones-es).

---

### 5. `Mark a message as read`
| Propiedad | Valor |
|---|---|
| Tipo | `n8n-nodes-base.gmail` |
| Operación | `markAsRead` |
| ID del mensaje | `={{ $('Gmail Nuevos Correos').item.json.id }}` |
| Credenciales | `gmailOAuth2` |

Marca el correo original como leído después de una subida exitosa. Esto funciona como señal implícita de éxito del workflow: solo los correos cuyos adjuntos se subieron completamente se marcan como leídos.

> ⚠️ Debido a que `Separar Adjuntos` genera un ítem por adjunto, este nodo se ejecuta dentro de un contexto multi-ítem. La expresión del Message ID **debe usar `.first()`**, no `.item`:
> ```
> {{ $('Gmail Nuevos Correos').first().json.id }}   ✓
> ```
> Usar `.item` provoca que n8n lance el error *"Can't determine which item to use"* cuando hay más de un adjunto. Consulta [Limitaciones](#limitaciones-es) para más detalles.

---

### 6. `No Operation, do nothing`
La rama `false` de `Es Factura`. Los correos que no son facturas se descartan silenciosamente — sin acción, sin notificación, sin registro. Permanecen **no leídos** en Gmail.

---

<a name="requisitos-es"></a>
## Requisitos y credenciales

### Credenciales requeridas en n8n

| Nombre de credencial | Tipo | Usada por | Cómo configurar |
|---|---|---|---|
| Gmail (tu cuenta) | `gmailOAuth2` | `Gmail Nuevos Correos`, `Mark a message as read` | [Docs Gmail n8n](https://docs.n8n.io/integrations/builtin/credentials/google/) |
| Google Drive (tu cuenta) | `googleDriveOAuth2Api` | `Subir a Google Drive` | Mismas credenciales OAuth2 de Google |

### Configurar la carpeta destino

1. Abre el nodo `Subir a Google Drive`
2. Conecta tu cuenta de Google Drive mediante OAuth2
3. En el campo **Folder**, usa el selector de carpetas para elegir o crear tu carpeta destino
4. El workflow subirá todos los adjuntos coincidentes a esa carpeta

### Permisos de Gmail requeridos

La credencial OAuth2 de Gmail necesita los siguientes scopes:
- `https://www.googleapis.com/auth/gmail.readonly` — para leer mensajes y adjuntos
- `https://www.googleapis.com/auth/gmail.modify` — para marcar mensajes como leídos

---

<a name="specs-es"></a>
## Especificaciones técnicas

| Componente | Tecnología |
|---|---|
| Motor de workflow | n8n (`executionOrder: v1`) |
| Fuente de correo | Gmail (OAuth2, trigger por polling diario) |
| Frecuencia de polling | Diario a las 00:00 |
| Detección de facturas | Regex sobre asunto + cuerpo (lógica OR) |
| Manejo de adjuntos | Fan-out en JavaScript (un ítem por adjunto) |
| Almacenamiento | Google Drive (carpeta configurable por el usuario) |
| Estado del workflow | **Activo** — corre continuamente |

---

<a name="diseno-es"></a>
## Decisiones de diseño

### ¿Por qué polling en lugar de trigger por push?
Gmail no soporta webhooks nativamente en n8n sin configuración adicional (Google Pub/Sub). El polling una vez al día a medianoche es un trade-off deliberado: elimina el uso continuo de la API a costa de hasta 24 horas de latencia entre la llegada del correo y la subida del archivo.

### ¿Por qué lógica OR en el filtro de facturas?
El filtro usa OR (asunto O cuerpo) para maximizar el recall — perder una factura real es peor que procesar un falso positivo. El trade-off es que correos de marketing o notificaciones bancarias que mencionen "factura" en el cuerpo pueden filtrarse. Cambia a lógica AND si los falsos positivos se convierten en un problema.

### ¿Por qué un nodo Code para separar los adjuntos?
El trigger de Gmail de n8n devuelve todos los adjuntos bajo un solo ítem como `attachment_0`, `attachment_1`, etc. No hay nodo nativo que haga fan-out de datos binarios de un ítem a múltiples ítems. El nodo Code es la solución estándar para este patrón.

### ¿Por qué marcar como leído al final?
Usar "marcar como leído" como paso final crea un mecanismo implícito de idempotencia: si el workflow falla a mitad de la ejecución (ej. error en la subida a Drive), el correo permanece no leído y será recogido en el siguiente ciclo de polling. No es una garantía — es una salvaguarda de mejor esfuerzo.

---

<a name="limitaciones-es"></a>
## Limitaciones y advertencias

### 🔴 Sin filtro de tipo de archivo
Todos los adjuntos se suben sin importar el formato. Esto significa que imágenes, archivos ZIP, documentos Word y hojas de cálculo de correos de facturas se almacenarán junto con los documentos fiscales reales.

**Impacto:** La carpeta `Facturas` acumula ruido — logos de empresas, firmas y archivos no relacionados embebidos en correos de facturas.

**Mitigación:** Agregar un filtro después de `Separar Adjuntos` que verifique `$json.fileName` contra extensiones permitidas:
```
{{ ['pdf', 'xml', 'zip'].includes($json.fileName.split('.').pop().toLowerCase()) }}
```
Para el contexto fiscal mexicano (CFDI), los formatos relevantes son `.xml` (el CFDI en sí) y `.pdf` (la representación legible por humanos).

---

### 🔴 Error de resolución de ítem en `markAsRead`
Debido a que `Separar Adjuntos` produce un ítem por adjunto, el nodo `Mark a message as read` se ejecuta dentro de un contexto multi-ítem. Si la expresión del Message ID usa `.item`:

```
{{ $('Gmail Nuevos Correos').item.json.id }}   ← fallará
```

n8n no puede determinar qué ítem parear y lanza:
> *"Can't determine which item to use"*

**Fix:** Reemplazar `.item` por `.first()` para resolver el correo original sin ambigüedad:

```
{{ $('Gmail Nuevos Correos').first().json.id }}   ✓
```

`.first()` siempre devuelve el correo fuente sin importar cuántos ítems de adjuntos estén en vuelo aguas abajo. Esto también resuelve las llamadas redundantes: dado que el mismo ID se marca como leído N veces (una por adjunto), usar `.first()` es tanto la expresión correcta como la segura.

---

### 🟡 Hasta 24 horas de latencia de procesamiento
El trigger diario se ejecuta una sola vez a medianoche. Una factura que llega a las 08:00 no se subirá hasta la ejecución de medianoche siguiente — hasta 16 horas después.

**Impacto:** Aceptable para flujos de contabilidad en lote y conciliaciones de fin de día. No aceptable si la carpeta de Drive alimenta un proceso aguas abajo que necesita las facturas disponibles minutos después de su recepción.

**Mitigación:** Si se necesita menor latencia, aumentar la frecuencia a cada hora o cada pocas horas sin impactar significativamente la cuota de la API de Gmail.

---

### 🟡 Falsos positivos en el filtro de facturas
La lógica OR significa que cualquier correo que contenga "factura" o "cfdi" en cualquier parte del cuerpo — incluyendo correos de marketing, notificaciones bancarias o newsletters — será enrutado a la rama de subida.

**Mitigación:** Cambiar el combinador a AND (asunto Y cuerpo deben coincidir), o agregar una lista de remitentes permitidos usando el campo de consulta de Gmail (`from:facturacion@proveedor.com`).

---

### 🟡 Los correos no-factura nunca se marcan como leídos
Los correos que no pasan el filtro `Es Factura` van a `NoOp` y permanecen no leídos. Serán re-evaluados en cada ciclo de polling posterior.

**Impacto:** Si Gmail tiene un gran backlog de correos no leídos con adjuntos que no son facturas, el workflow los procesará y descartará repetidamente, consumiendo cuota de API innecesariamente.

**Mitigación:** Marcar todos los correos procesados como leídos independientemente del resultado del filtro — o usar una etiqueta de Gmail en lugar del estado leído/no leído como indicador de procesamiento.

---

### 🟡 Carpeta destino hardcodeada
El ID de la carpeta de Drive está embebido en la configuración del nodo. No hay parametrización.

**Mitigación:** Usar el selector de carpetas de Drive para elegir tu propia carpeta al configurar el workflow.

---

### 🟡 Sin manejo de errores
Si la subida a Drive falla (cuota excedida, error de permisos, timeout de red), el workflow se detiene silenciosamente. El correo permanece no leído, por lo que se reintentará en el siguiente poll — pero no hay notificación, log ni cola de mensajes fallidos.

---

<a name="casos-es"></a>
## Casos de uso recomendados

**Adecuado para:**
- ✅ Recolección de facturas personal o para pequeñas empresas (bajo volumen de correos)
- ✅ Centralizar documentos fiscales de múltiples remitentes en una sola carpeta de Drive
- ✅ Reducir trabajo manual de descarga y subida para flujos de contabilidad
- ✅ Primer paso en un pipeline más amplio (ej. Drive → software de contabilidad)

**No adecuado para:**
- ❌ Bandejas de entrada de alto volumen donde los correos con adjuntos que no son facturas son frecuentes
- ❌ Entornos donde el control del tipo de archivo es crítico (sin el filtro de extensiones)
- ❌ Configuraciones multi-cuenta (una instancia del workflow por cuenta de Gmail)
- ❌ Entornos de producción que requieren auditoría o notificaciones de error

---

<a name="extensiones-es"></a>
## Ideas de extensión

| Mejora | Enfoque |
|---|---|
| Filtro de tipo de archivo | Agregar nodo `IF` después de `Separar Adjuntos` para permitir solo `.pdf` y `.xml` |
| Lista de remitentes permitidos | Restringir la consulta del trigger de Gmail a remitentes conocidos de facturas (`from:...`) |
| Enrutamiento de carpeta por remitente | Usar nodo `Switch` para enrutar a diferentes carpetas de Drive por dominio del remitente |
| Parseo de XML CFDI | Agregar nodo Code para extraer RFC, monto y UUID del XML antes de subir |
| Log en hoja de cálculo | Agregar los metadatos del correo y nombre del archivo a un Google Sheet en cada subida exitosa |
| Notificaciones de error | Agregar rama de error → alerta por Slack o correo en caso de falla en la subida a Drive |
| Reducir `markAsRead` redundantes | Recolectar IDs de correo después del split, deduplicar y marcar una sola vez por correo |
| Etiqueta en lugar de estado de lectura | Usar etiquetas de Gmail como indicadores de procesamiento para una idempotencia más robusta |

---

## Estructura de archivos

```
n8n-agent-flows/
└── process-automation/
    └── invoice-uploader/
        ├── invoice-uploader.json     # Exportable n8n workflow / Workflow exportable de n8n
        └── README.md                 # This file / Este archivo
```

---

*Part of the [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) repository — a collection of production-grade n8n workflows for AI systems, RAG pipelines, and process automation.*

*Parte del repositorio [`n8n-agent-flows`](https://github.com/DarioArteaga/n8n-agent-flows) — una colección de workflows de n8n para sistemas AI, pipelines RAG y automatización de procesos.*