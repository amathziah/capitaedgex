# CapitaEdgeX

**A GST-compliant invoicing and finance workspace for Indian businesses, with an agentic AI layer over live financial data.**

🔗 **Live product: [capitaedgex.com](https://capitaedgex.com/)**

**Application repositories:** [Frontend](https://github.com/amathziah/capita-edge-x-frontend) · [Backend](https://github.com/amathziah/capita-edge-x-backend)

> **About this repository.** CapitaEdgeX is a commercial product and the application source is
> maintained in the public frontend and backend repositories linked above. This repo is the public
> technical record: the architecture, the design decisions worth defending, and the parts of the
> codebase that carry no commercial sensitivity (the GST/SAC tax engine, which is built on public
> tax data and included here in full).

![CapitaEdgeX](docs/product.png)

---

## What it does

Create GST-compliant invoices, track payments, manage expenses, and get tax-deadline reminders.
On top of that sits an AI layer that does not merely answer questions about the data — it runs
multi-step workflows against it: **collections sweep**, **month-end close**, **cash-shortfall
response**.

## Architecture

Full write-up: **[docs/ai-architecture.md](docs/ai-architecture.md)**

The backend uses a bounded agentic workflow. An LLM intent router can select a registered multi-step skill; other requests enter Gemini's function-calling loop, where the model can call only tools declared by the backend.

```mermaid
flowchart TD
    USER[User request] --> AUTH[Auth, rate limit, context]
    AUTH --> ORCH[AI orchestrator]
    ORCH --> CACHE{Non-streaming chat cache hit?}
    CACHE -->|Yes| RESPONSE[Return cached response]
    CACHE -->|No| INTENT[LLM intent router]
    INTENT --> ROUTE{Known skill with confidence at least 0.7?}
    ROUTE -->|Yes| SKILL[Registered skill playbook]
    ROUTE -->|No| LOOP[Gemini function-calling loop, up to 8 tool rounds]
    SKILL --> TOOLS[Registered tool set]
    LOOP -->|Function call| TOOLS
    TOOLS --> DATA[Read model, knowledge retrieval, or action executor]
    DATA -->|Tool result| LOOP
    LOOP -->|Final text| CRITIC[Advisory critic on tool-loop path]
    SKILL --> RESULT[Prepare result]
    CRITIC --> RESULT
    RESULT --> STORE[Usage, trace, conversation persistence]
    STORE --> RESPONSE[Return response]
```

### Layer-by-layer

| Layer | Input | Responsibility | Output to next layer |
| :--- | :--- | :--- | :--- |
| **1. Ingress and auth** | HTTP request and bearer token | Resolve user and organization; apply AI rate limit. | Authenticated request with organization context. |
| **2. Controller/context** | Query, persona, history, conversation ID | Validate query; load organization settings and assemble currency, business, and temporal context. | Query plus AI context. |
| **3. Session/cache** | Query and organization | Get/create in-process session; check normalized organization/query cache; create trace on cache miss. | Cached response or a traced request. |
| **4. Intent router** | Query, recent caller context, skill descriptions | Use Gemini JSON classification with heuristic fallback; validate skill ID and confidence. | Skill route at confidence `>= 0.7`, or regular tool-loop route. |
| **5. Orchestrator** | Intent and context | Run the selected skill playbook or Gemini function-calling loop (maximum 8 tool rounds). | Skill result, or final answer and tool evidence. |
| **6. Tool registry** | Declared Gemini function call or skill step | Dispatch allowed tools to business reads, knowledge retrieval, analytics, or action execution. | `{ data, meta }` result or a tool error. |
| **7. Data/action layer** | Tool arguments and organization ID | Read model returns tenant-scoped data; RAG returns knowledge chunks; action executor can perform registered side effects. | Evidence, source metadata, or action result. |
| **8. Critic** | Final answer and successful tool results | Advisory check for unsupported claims on the regular tool-loop path. | Verdict metadata; original answer remains unchanged. |
| **9. Persistence/response** | Result, usage, trace ID | Store usage, conversation messages, and trace records; return JSON or SSE completion. | Final response. |

The cache branch applies to `POST /api/ai/chat`; `POST /api/ai/chat/stream` follows the same intent/skill/tool architecture but bypasses this chat-cache lookup. The full guide expands each layer's inputs, outputs, and failure behavior and includes a sequence diagram: **[Layer-by-layer AI architecture](docs/ai-architecture.md#layer-by-layer-execution)**.

### Hybrid RAG for business knowledge

```mermaid
flowchart LR
    SOURCE[(Organization business knowledge)] --> CHUNK[Chunk and embed on reindex]
    CHUNK --> INDEX[(Postgres chunks and pgvector index)]
    QUERY[Knowledge question] --> QEMBED[Gemini query embedding]
    QEMBED --> SEARCH[Hybrid retrieval RPC]
    INDEX --> SEARCH
    SEARCH --> VECTOR[Cosine similarity]
    SEARCH --> KEYWORD[Postgres full-text ranking]
    VECTOR --> FUSE[Reciprocal-rank fusion]
    KEYWORD --> FUSE
    FUSE --> EVIDENCE[Source excerpts and metadata]
    EVIDENCE --> MODEL[Gemini answer with retrieved evidence]
```

The `get_knowledge_base` tool exposes hybrid retrieval for organization knowledge. Chunks are embedded with Gemini `gemini-embedding-001` and ranked using pgvector cosine similarity plus Postgres full-text search. If vector search is unavailable or has no matches, the backend falls back to organization-scoped keyword matching. Existing entries require the backend migration and a reindex operation before they are in the vector index.

---

## Design choices and current limits

### 1. Tax rules are deterministic; the assistant is grounded in retrieved data

The public tax engine contains deterministic GSTIN validation, state mapping, and intra- versus inter-state tax classification. The AI backend retrieves live business facts through organization-scoped tools and instructs Gemini to ground user-data claims in tool evidence. The model still writes the natural-language answer; the critic is advisory, not a guarantee.

The tax engine is included in this repo: [`src/tax/`](src/tax/).

### 2. A critic flags unsupported claims on one path

A second, cheaper model pass checks the regular function-calling answer against successful tool
results for unsupported numbers, names, dates, and claims about user data.

The critic does not run for skill workflows, and it does not block or revise the original answer.
It **fails open** if unavailable or malformed. Treat it as an additional signal, not a safety
boundary or a guarantee of hallucination-free output.

### 3. Bounded workflows with explicit operational limits

- **Defined tool surface and fixed skills** — the model can only invoke declared functions; skills encode repeatable multi-step playbooks.
- **Model fallback chain and process-local rate bucket** — retry and cooldown behavior are implemented in the backend runtime.
- **Short-term chat caching and traces** — chat responses are cached by organization/query and orchestration stages are persisted.
- **Approval is not universal** — Collections Sweep pauses before its drafts are sent, but mutation-capable tools in the regular tool loop can execute directly.
- **Plan budgets are not currently enforced** — the budget service reports unlimited usage, although usage is logged.
- **Model complexity routing is not currently active** — the router's model selection currently returns `gemini-2.5-flash` regardless of detected complexity.

For endpoint setup, module paths, and the full operational flow, see [the AI architecture guide](docs/ai-architecture.md) and the linked [backend source repository](https://github.com/amathziah/capita-edge-x-backend).

---

## The tax engine (included source)

Real, runnable code with no commercial sensitivity — it encodes public Indian tax data.

**[`src/tax/gstUtils.js`](src/tax/gstUtils.js)**
GSTIN format validation, state-code mapping for all states and UTs, and intra- vs inter-state
detection that decides CGST+SGST versus IGST.

**[`src/tax/sacDatabase.js`](src/tax/sacDatabase.js)**
SAC (Service Accounting Code) catalogue mapped to GST rates and TDS sections, with
auto-suggestion from an item description — so a user writing "logo design" gets the right code,
rate and TDS section without knowing any of them.

```js
import { validateGstin, getGstType } from './src/tax/gstUtils.js';

validateGstin('29ABCDE1234F1Z5');        // → true
getGstType('29ABCDE…', '29ABCDE…');      // → intra-state: CGST + SGST
getGstType('29ABCDE…', '27ABCDE…');      // → inter-state: IGST
```

---

## Stack

**Backend** Node.js · Express · PostgreSQL (Supabase) · Redis · `decimal.js` · PDFKit · Nodemailer
**Frontend** React · Vite · Tailwind
**AI** Google Gemini — orchestrator, tool registry, critic, model router
**Infra** Docker · Vercel (web) · Render (API) · CI secret scanning

## Author

**Amathziah J** — [LinkedIn](https://www.linkedin.com/in/amathziah-j-832646214/) · [GitHub](https://github.com/amathziah)
