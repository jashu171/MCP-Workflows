# MCP – Explorium Firmographics Enrichment (Google Sheets → AI Agent → Sheets)

Beginner-friendly guide to build an **n8n** workflow that enriches firmographics using an **AI Agent** with **Explorium MCP** tools and writes results back to the **same Google Sheet**—fully automatic.

> ⚠️ **Do not “fix” header spellings** in your sheet. The mapping depends on them **exactly as written** below.

---

## Table of Contents
- [Goal](#goal)
- [What You’ll Build](#what-youll-build)
- [Prerequisites](#prerequisites)
  - [Required Accounts & Access](#required-accounts--access)
  - [API Keys & Credentials in n8n](#api-keys--credentials-in-n8n)
  - [OAuth Quick Note](#oauth-quick-note)
  - [MCP Client Setup Options (Explorium)](#mcp-client-setup-options-explorium)
- [Canvas Wiring (One-Line View)](#canvas-wiring-one-line-view)
- [Workflow Steps (Node-by-Node, Copy-Paste)](#workflow-steps-node-by-node-copy-paste)
  - [Node 1 — Google Sheets Trigger](#node-1--google-sheets-trigger-trigger)
  - [Node 2 — IF (Filter Valid Rows)](#node-2--if-filter-valid-rows-if)
  - [Node 3 — Split in Batches (Loop)](#node-3--split-in-batches-loop-split-in-batches)
  - [Node 4 — Simple Memory](#node-4--simple-memory-ai-memory)
  - [Node 5 — MCP Client1](#node-5--mcp-client1-mcp-client--stdio--http--sse)
  - [Node 6 — Google Gemini Chat Model](#node-6--google-gemini-chat-model-ai-model)
  - [Node 7 — Output Parser](#node-7--output-parser-structured-output-parser)
  - [Node 8 — AI Agent1](#node-8--ai-agent1-ai-agent--langchain)
  - [Node 9 — Code (Format Output)](#node-9--code-format-output-code)
  - [Node 10 — Google Sheets (Append/Update)](#node-10--google-sheets-appendupdate-google-sheets)
  - [Loop Back — Continue the Batch](#loop-back--continue-the-batch)
- [Column Mapping (Exact Headers)](#column-mapping-exact-headers)
- [Example Input & Output](#example-input--output)
- [Features](#features)
- [Dependencies](#dependencies)
- [Configuration Notes & Tips](#configuration-notes--tips)
- [Security Notes](#security-notes)
- [Links](#links)
- [Contributors](#contributors)
- [License](#license)

---

## Goal
When you add or edit a row in a Google Sheet, this workflow uses an **AI Agent** with **Explorium MCP** tools to:
1) find the company’s **business_id** and  
2) enrich **firmographics** (NAICS, employee range, annual revenue range),  
then writes the results back to the **same sheet**.

---

## What You’ll Build
A drag-and-drop **n8n** automation:

```
[Google Sheets Trigger]
      → [IF (Filter Valid Rows)]
      → [Split in Batches (Loop)]
      → [Simple Memory]
      → [AI Agent (Gemini + MCP + Output Parser)]
      → [Code (Format Output)]
      → [Google Sheets (Append/Update)]
      → (loop back to Split in Batches)
```

**Canvas view (reference):**  
![n8n workflow canvas showing nodes and loop](./canvas.png)

---

## Prerequisites

### Required Accounts & Access
- **n8n** (Cloud or self-hosted) with **AI nodes enabled**.
- **Google Account** with access to your Google Sheet.
- A spreadsheet containing a sheet (tab) named **`Explorium DATA`** with **exact headers** (typos *intentional*):
  - `comapny`  
  - `website URL`  
  - `Bussiness ID`  
  - `Naics`  
  - `No Of Employees Range`  
  - `Yearly revenue`
- **Google Gemini API** access (API key via **Google AI Studio**).
- **Explorium MCP** available to the Agent (via one of the client options below).

> ⚠️ Do not change header spellings. They must match **exactly**.

**Example Sheet (with exact headers):**  
![Google Sheet with required headers and sample rows](./sheet-data.png)

### API Keys & Credentials in n8n
- **Google Sheets (OAuth2)**  
  *Settings → Credentials → Google Sheets OAuth2 API → Create & Authorize.*
- **Google Gemini / PaLM (API Key)**  
  *Settings → Credentials → Google Palm API → Paste API key from Google AI Studio.*
- **MCP Client (choose one)**  
  *Settings → Credentials → MCP Client (STDIO / HTTP / SSE) → Configure per your chosen option.*

### OAuth Quick Note
OAuth lets n8n access your Google Sheets without storing your password.  
**Callback URL (exact format):**
```
https://your-n8n-instance.com/rest/oauth2-credential/callback
```

### MCP Client Setup Options (Explorium)

**Option A — STDIO via NPX (community MCP)**  
- Explorium MCP GitHub: https://github.com/explorium-ai/mcp-explorium  
- Use **NPX** with the community **mcp-remote** to bridge stdio → Explorium MCP endpoint.  
- n8n **“MCP Client (STDIO)”** credential:
  - **Command:** `npx`
  - **Arguments:** `-y mcp-remote https://mcp.explorium.ai/mcp`

**Credential example (screenshot):**  
![n8n MCP Client (STDIO) credential using npx mcp-remote](./MCP-config.png)

- Equivalent JSON config (reference):
```json
{
  "mcpServers": {
    "explorium": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://mcp.explorium.ai/mcp"]
    }
  }
}
```
- After saving, **test** the credential once so the Agent can discover tools like:
  - `match-business`
  - `enrich-businesses-firmographics`

**Option B — HTTP or SSE MCP Client (API key required)**  
- Get an API key in **Explorium Integrations**: https://admin.explorium.ai/integrations  
- In n8n, create **MCP Client (HTTP)** *or* **MCP Client (SSE)** pointing to the Explorium MCP endpoint and provide the API key (field or headers per node UI).  
- Keep the key secure: use **n8n Credentials** (encrypted at rest), not inline strings in nodes.

---

## Canvas Wiring (One-Line View)
`[Google Sheets Trigger] → [IF (Filter Valid Rows)] → [Split in Batches (Loop)] → [Simple Memory] → [AI Agent (Gemini + MCP + Output Parser)] → [Code (Format Output)] → [Google Sheets (Append/Update)] → (loop back to Split in Batches)`

---

## Workflow Steps (Node-by-Node, Copy-Paste)

### Node 1 — Google Sheets Trigger (Trigger)
**Purpose:** Start when rows are created/edited in the target sheet.  
**Drag & Drop:** Search “**Google Sheets Trigger**” → drag → rename **`Google Sheets Trigger`**.  
**Configure:**
- **Mode/Operation:** *Trigger on changes via polling*
- **Document:** Select your spreadsheet (browse or paste ID).
- **Sheet:** `Explorium DATA`
- **Include In Output:** *Both*
- **Polling Interval:** e.g., *1 minute*.

**Test:** Activate workflow, then add/edit a row—an execution should appear.

---

### Node 2 — IF (Filter Valid Rows) (IF)
**Purpose:** Only continue when both a company name and website are present.  
**Drag & Drop:** Search “**IF**” → drag → rename **`Filter Valid Rows`**.  
**Connect:** `Google Sheets Trigger → Filter Valid Rows`.  
**Configure:**
- **Combinator:** *AND*  
- **Case Sensitive:** *true*  
- **Type Validation:** *strict*  
- **Conditions (Left → Operator → Right):**
  - `={{ $json.comapny }}  → is not empty`
  - `={{ $json['website URL'] }} → is not empty`

**Routes:**  
- **True:** continue  
- **False:** stop (row is incomplete)

**Copy-paste expressions:**
```
{{ $json.comapny }}
{{ $json['website URL'] }}
```

---

### Node 3 — Split in Batches (Loop) (Split In Batches)
**Purpose:** Process rows one-by-one; failures don’t block others.  
**Drag & Drop:** Search “**Split in Batches**” → drag → rename **`Loop Over Items`**.  
**Connect:** `Filter Valid Rows (true) → Loop Over Items`.  
**Configure:**
- **Batch Size:** `1`

> After the Google Sheets update later, you’ll connect the **Continue** output back to this node to iterate.

---

### Node 4 — Simple Memory (AI Memory)
**Purpose:** Keep short, per-company memory during one item’s processing.  
**Drag & Drop:** AI category → “**Simple Memory**” → rename **`Simple Memory`**.  
**Connect:** `Loop Over Items → Simple Memory`.  
**Configure:**
- **Mode/Operation:** *Buffer window*  
- **Session ID Type:** *Custom key*  
- **Session Key (Expression):**
  ```
  {{ $json.comapny }}
  ```
- **Context Window Length:** `50`

---

### Node 5 — MCP Client1 (MCP Client — STDIO / HTTP / SSE)
**Purpose:** Provide the Explorium tools for the agent to call.  
**Drag & Drop:** Your chosen **MCP Client** node → rename **`MCP Client1`**.  
**Connect:** *Not inline; attached as a Tool to the Agent.*  
**Configure Credential:**
- **STDIO (NPX):** as in *Option A*  
- **HTTP/SSE:** as in *Option B* (provide API key)

---

### Node 6 — Google Gemini Chat Model (AI Model)
**Purpose:** Underlying chat model for reasoning & tool selection.  
**Drag & Drop:** “**Google Gemini Chat Model**” → rename **`Gemini (Agent)`**.  
**Connect:** *Attached to the Agent.*  
**Configure:**
- **Model:** `models/gemini-2.5-pro`
- **Temperature:** `0.4`
- **Max Tokens:** *leave default*

---

### Node 7 — Output Parser (Structured Output Parser)
**Purpose:** Force the AI to return strict JSON we can map to the sheet.  
**Drag & Drop:** “**Structured Output Parser**” → rename **`Output Parser`**.  
**Connect:** *Attach to the Agent as its output format.*  
**Schema Example (copy-paste):**
```json
{
  "name": "Microsoft Corporation",
  "website": "https://www.microsoft.com",
  "business_id": "a34bacf839b923770b2c360eefa26748",
  "naics": "511210",
  "number_of_employees_range": "10001+",
  "yearly_revenue_range": "100B-1T"
}
```

**Parser’s Model (internal):**
- Add a second **Google Gemini Chat Model**:
  - **Model:** `models/gemini-2.5-pro`
  - **Temperature:** `0.4`

---

### Node 8 — AI Agent1 (AI Agent — LangChain)
**Purpose:** Orchestrate the task:  
1) Call `match-business` to get **business_id**, then  
2) Call `enrich-businesses-firmographics`, then  
3) Return **structured JSON** via the Output Parser.

**Drag & Drop:** “**AI Agent (LangChain)**” → rename **`AI Agent1`**.  
**Connect:** `Simple Memory → AI Agent1`.  
**Attach inside Agent:**
- **Language Model:** `Gemini (Agent)` (Node 6)
- **Memory:** `Simple Memory` (Node 4)
- **Tools:** `MCP Client1` (Node 5)
- **Output Parser:** `Output Parser` (Node 7)

**Prompts**

**User Prompt (Text):**
```
You will be provided with company information including:

Name: {{ $json.comapny }}
Domain: {{ $json['website URL'] }}
```

**System Prompt (full):**
```
Your task is to research and extract the following key business metrics for this company:
Note : Reduce the Token Size below 6000k tokens
Required Data Points:
- Name (keep the same as provided)
- Website (keep the same as provided)
- Annual revenue (as business_id first, then get the range)
- Number of employees (range)
- NAICS code

Use the Explorium tools to:
1. First use match-business to find the business ID
2. Then use enrich-businesses-firmographics to get the company details

Return the data exactly in this format.

Structured Output (what the Agent must yield):
{
  "name": "string - company name",
  "website": "string - domain url",
  "business_id": "string - Explorium business id",
  "naics": "string - NAICS code",
  "number_of_employees_range": "string - e.g., 1-10, 11-50, 10001+",
  "yearly_revenue_range": "string - e.g., 1M-10M, 100B-1T"
}
```

---

### Node 9 — Code (Format Output) (Code)
**Purpose:** Normalize the agent’s output and provide safe fallbacks.  
**Drag & Drop:** “**Code**” → rename **`Format Output`**.  
**Connect:** `AI Agent1 → Format Output`.  
**Code (copy-paste):**
```js
// WHAT THIS CODE DOES:
// - Maps the AI Agent's structured output to a flat object
// - Falls back to original inputs when fields are missing
// INPUT:  $json from AI Agent (with $json.output.*)
// OUTPUT: { name, website, business_id, naics, number_of_employees_range, yearly_revenue_range }

return {
  name: $json.output?.name || $json.name || $json.comapny,
  website: $json.output?.website || $json.website || $json['website URL'],
  business_id: $json.output?.business_id || '',
  naics: $json.output?.naics || '',
  number_of_employees_range: $json.output?.number_of_employees_range || '',
  yearly_revenue_range: $json.output?.yearly_revenue_range || ''
};

// BREAKDOWN:
// - Use optional chaining (?.) to avoid crashes if output is missing
// - Prefer AI output; otherwise fall back to trigger values
```

---

### Node 10 — Google Sheets (Append/Update) (Google Sheets)
**Purpose:** Write enrichment back to the sheet, updating if the row exists (matching `comapny`), otherwise appending.  
**Drag & Drop:** “**Google Sheets**” → rename **`Append/Update Company`**.  
**Connect:** `Format Output → Append/Update Company`.  
**Configure:**
- **Operation:** *Append or update*
- **Document:** Select same spreadsheet as trigger.
- **Sheet:** `Explorium DATA`
- **Columns → Mapping Mode:** *Define below*
- **Matching Columns:** `comapny`
- **Column Value Mappings (copy-paste exactly):**
```
comapny                    = {{ $json.name }}
website URL                = {{ $json.website }}
Bussiness ID               = {{ $json.business_id }}
Naics                      = {{ $json.naics }}
No Of Employees Range      = {{ $json.number_of_employees_range }}
Yearly revenue             = {{ $json.yearly_revenue_range }}
```

---

### Loop Back — Continue the Batch
After **Append/Update Company**, connect its **Continue** (or equivalent loop output) back to **`Loop Over Items`** to process the next row in the batch.

**Final Checks**
- Activate the workflow.
- Add/edit a row with valid **`comapny`** and **`website URL`**.
- Watch Executions; confirm enrichment fields are filled.

---

## Column Mapping (Exact Headers)

| Sheet Column (exact)        | Filled From                                   |
|-----------------------------|-----------------------------------------------|
| `comapny`                   | `{{ $json.name }}`                            |
| `website URL`               | `{{ $json.website }}`                         |
| `Bussiness ID`              | `{{ $json.business_id }}`                     |
| `Naics`                     | `{{ $json.naics }}`                           |
| `No Of Employees Range`     | `{{ $json.number_of_employees_range }}`       |
| `Yearly revenue`            | `{{ $json.yearly_revenue_range }}`            |

> ⚠️ Keep spellings exactly as above.

---

## Example Input & Output

**Before (you type/add/edit):**
```
comapny: Contoso Ltd
website URL: https://www.contoso.com
Bussiness ID:
Naics:
No Of Employees Range:
Yearly revenue:
```

**After (workflow fills):**
```
comapny: Contoso Ltd
website URL: https://www.contoso.com
Bussiness ID: a34bacf839b923770b2c360eefa26748
Naics: 511210
No Of Employees Range: 10001+
Yearly revenue: 100B-1T
```

*(Values are illustrative; your results come from Explorium.)*

---

## Features
- Zero-touch enrichment on **add/edit** of rows.
- **Strict JSON** output via parser to keep columns clean.
- **Batch-safe loop**: one bad item won’t block others.
- **Append or update** behavior to avoid duplicates.
- Works with **STDIO (NPX)** or **HTTP/SSE** MCP clients.

---

## Dependencies
- **n8n** (Cloud or self-hosted) with **AI nodes**.
- **Google Sheets** (OAuth2).
- **Google Gemini** `models/gemini-2.5-pro` (API key).
- **Explorium MCP** tools:
  - `match-business`
  - `enrich-businesses-firmographics`

---

## Configuration Notes & Tips
- **Polling interval**: 1–5 minutes is common. Lower intervals = more API calls.
- **Session Memory**: Using `{{ $json.comapny }}` isolates memory per company during the loop.
- **Parser**: Keep the schema concise; downstream mapping assumes those six fields.
- **Costs/quotas**: Gemini + Explorium usage incurs API costs—monitor in their dashboards.
- **Sheet access**: Ensure the OAuth’d Google account can read/write the spreadsheet.

---

## Security Notes
- Store **all API keys** in **n8n Credentials**, not inside nodes.
- Restrict access to your n8n instance and credential editing.
- If using HTTP/SSE MCP, prefer **header-based** API key injection via the credential form.

---

## Links
- **Explorium MCP GitHub (NPX/STDIO option):** https://github.com/explorium-ai/mcp-explorium  
- **Explorium Integrations (API key for HTTP/SSE client):** https://admin.explorium.ai/integrations

---

## Contributors
Add yourself and reviewers here:
- Your Name (@handle) — author

---

## License
Add your license here (e.g., MIT).
