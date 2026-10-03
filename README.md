# CapitaEdgeX

**A GST-compliant invoicing and finance workspace for Indian businesses, with an agentic AI layer over live financial data.**

🔗 **Live product: [capitaedgex.com](https://capitaedgex.com/)**

> **About this repository.** CapitaEdgeX is a commercial product and the application source is
> private. This repo is the public technical record: the architecture, the design decisions worth
> defending, and the parts of the codebase that carry no commercial sensitivity (the GST/SAC tax
> engine, which is built on public tax data and included here in full).
> **Read access to the private source is available on request** — open an issue or reach me on
> [LinkedIn](https://www.linkedin.com/in/amathziah-j-832646214/).

![CapitaEdgeX](docs/product.png)

---

## What it does

Create GST-compliant invoices, track payments, manage expenses, and get tax-deadline reminders.
On top of that sits an AI layer that does not merely answer questions about the data — it runs
multi-step workflows against it: **collections sweep**, **month-end close**, **cash-shortfall
response**.

## Architecture

Full write-up: **[docs/ai-architecture.md](docs/ai-architecture.md)**

```
Request → Intent router → Orchestrator ─┬→ Tool registry → Read model → Postgres
                                        ├→ Action executor (writes, guarded)
                                        └→ Skills (collections sweep, month-end close, …)
                                                  ↓
                                        Critic (verifies grounding)
                                                  ↓
                                              Response
```

| Layer | Responsibility |
| :--- | :--- |
| **Intent router** | Classifies the request and selects a path |
| **Orchestrator** | Plans and runs the tool/action sequence |
| **Tool registry** | Declares every capability the model may invoke |
| **Read model** | Serves query-shaped views of financial data |
| **Action executor** | Performs state changes, separately from reads |
| **Critic** | Verifies the final answer against the tool results |
| **Trace** | Per-request record of model calls, tools and cost |

---

## Three decisions worth defending

### 1. Money never goes through the model

Every figure a user sees is computed in deterministic libraries using **exact decimal
arithmetic** — never floating point, and never inferred by an LLM. GST type (CGST+SGST vs IGST),
tax rates, TDS sections and totals are pure functions of the invoice data.

The model interprets and explains. It does not calculate. That keeps every number reproducible
and auditable, which is non-negotiable in anything touching tax.

The tax engine is included in this repo: [`src/tax/`](src/tax/).

### 2. A critic verifies the model against its own evidence

A second, cheaper model pass checks whether the final answer is actually grounded in the tool
results returned that turn — catching hallucinated figures, names and dates that prompt-level
grounding rules miss.

It runs in JSON mode with capped tokens, skips turns with no tool calls (nothing to verify
against), and **fails open**: if the critic itself errors, the user still gets their answer.
Verification must never become a new way for the product to break.

### 3. Built for cost and failure, not just the happy path

- **Model routing by query complexity** — simple lookups do not pay for a reasoning model.
- **Rate-limit-aware fallback chain** — ordered by real provider limits, not guesswork.
- **Circuit breaker** (`opossum`) — stops hammering a failing provider.
- **Redis caching** and **per-request tracing** — bounded cost, visible spend.
- **Concurrency-tested invoice writes** — financial state must survive simultaneous updates.

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
