# LeadForge — Automated Lead Enrichment Pipeline

An n8n workflow that turns a messy list of raw leads into a clean, AI-scored,
sortable one. It reads leads from a Google Sheet, fetches each company's public
website, strips it to clean text, and — **only if that text is real** — has
Google Gemini summarise and score the company against a configurable Ideal
Customer Profile (ICP), then writes the result back to the same sheet.

It is **general-purpose**: the ICP is a single editable block, so the same
pipeline scores leads for any business. It ships demoed with an AI-consultancy
ICP.

> Portfolio project 2. Design docs: [`PROJECT.md`](PROJECT.md) (what/who),
> [`ARCHITECTURE.md`](ARCHITECTURE.md) (deep reference),
> [`ARCHITECTURE-ESSENTIALS.md`](ARCHITECTURE-ESSENTIALS.md) (one-page scan).

---

## What makes it credible

- **Grounded by construction — substantially reduced hallucination risk.** The
  LLM is only ever called for a lead we actually fetched text for. Three
  deterministic gates run in front of it (see below). This closes off the
  worst-case fabrication paths — empty and unfetchable sites — though it is not a
  total guarantee (see the honest limitation on gate 3 below).
- **Structured output.** Gemini is called with native `responseSchema`, so
  results come back as guaranteed JSON — no "please return JSON" prompting and
  no parse-retry loop.
- **Graceful failure.** Any single lead failing (missing site, dead host, thin
  content, quota error) routes to a "needs manual review" fallback that records
  *why*, and never crashes the run. Rows already enriched stay written.
- **Configurable.** Swap one ICP block to retarget the tool.

## The grounding gates (the core anti-hallucination discipline)

Every gate is deterministic and runs **before any token is spent**. Each writes
its reason into the `review_reason` column so you can see the discipline working.

| Gate | Fires when | `review_reason` |
|---|---|---|
| 0 — No company name | the row's `company_name` is blank | `no_company_name` |
| 1 — No website | the row's `website` is blank | `no_website` |
| 2 — Fetch failed | dead host, timeout, 403 bot-wall, non-2xx | `fetch_failed` |
| **3 — Thin content** | **HTTP 200 but too little text after stripping HTML** | `thin_content` |

**Gate 3 is the point.** An HTTP 200 is *not* evidence — JS-only single-page
apps, bot-walls, parked domains and cookie interstitials all return 200 with no
company information. Without this gate the model would be handed an empty string
and would invent a plausible company. Gates 1–2 are hygiene; gate 3 is what
substantially reduces hallucination risk on the cases that matter most.

**Honest limitation — gate 3 is a length heuristic, not a semantic check.** It
tests whether *enough clean text* came back (`MIN_CLEAN_CHARS`), not whether that
text is *actually about the company*. A page with 400 characters of generic
boilerplate ("Welcome to our website — cookies help us…") passes the gate, and
the model may then over-read it. So the honest claim is **"substantially reduces
hallucination risk, especially for empty or unfetchable sites"** — not "never
fabricates." A **semantic relevance check** (does the fetched text mention the
company / contain real business signals?) is a documented Phase-2 improvement.

## Data model

**Input row** (Google Sheet, one lead per row):

| column | required | notes |
|---|---|---|
| `company_name` | yes | |
| `website` | no | may be blank → gate 1 |
| `email` | no | reserved for the Phase-2 agentic branch |

**Output columns** (written back to the same row):

`summary`, `industry`, `fit_score` (1–10), `recommended_action`, `enriched_at`,
`status` (`enriched` \| `needs_manual_review`), `review_reason`.

Create these header columns in the sheet before the first run (plus a blank
`status` column — its emptiness is what the skip-already-enriched gate reads).

### Sample lead list

[`leadforge-leads.csv`](leadforge-leads.csv) is a ready-to-import 8-lead demo
list with the full 10-column header. It is deliberately built so **every gate
visibly fires** — not just the easy blank-website one. Import it into Google
Sheets (**File → Import → Upload**) and run.

| Lead | Website | Expected outcome |
|---|---|---|
| Basecamp | basecamp.com | ✅ enriched |
| Zapier | zapier.com | ✅ enriched — also exercises the size cap |
| Buffer | buffer.com | ✅ enriched |
| Atlassian | www.atlassian.com | ✅ enriched — also shows why stripping matters |
| Harbourline Freight | *(blank)* | ⛔ **gate 1** → `no_website` |
| Meridian Logistics | example.com | ⛔ **gate 3** → `thin_content` (parked-domain-style) |
| G2 | www.g2.com | ⛔ **gate 2** → `fetch_failed` (403 bot-wall) |
| Excalidraw | excalidraw.com | ⛔ **gate 3** → `thin_content` (JS-only SPA shell) |

**These outcomes were measured, not guessed.** Fetched with the workflow's exact
User-Agent:

| URL | Status | Raw HTML | Clean text after stripping |
|---|---|---|---|
| basecamp.com | 200 | 58 KB | 5,395 chars |
| zapier.com | 200 | 588 KB | 13,518 chars (capped to 8,000) |
| buffer.com | 200 | 604 KB | 4,169 chars |
| www.atlassian.com | 200 | **2.6 MB** | 4,443 chars |
| example.com | 200 | 559 B | **127 chars** → under threshold |
| www.g2.com | **403** | — | — |
| excalidraw.com | 200 | 6.8 KB | **0 chars** → SPA shell |

Two things this table demonstrates on its own:

- **Atlassian: 2.6 MB of HTML → 4.4 KB of text.** A 99.8% reduction. Sending the
  raw page would have burned enormous token budget for the same information —
  this is why the HTML-stripping node exists.
- **Excalidraw returns HTTP 200 and zero usable text.** A success status with no
  evidence whatsoever. This is exactly the case gate 3 exists to catch, and
  exactly what an HTTP-error-only check would have waved through into the model.

> ⚠️ Live sites drift. A 403 wall can lift, an SPA can adopt server-side
> rendering, and a parked domain can get a real page. Re-check these if the demo
> stops showing what you expect — the gates are correct regardless; only which
> *row* trips which gate can change.

> **Write-back key:** the workflow matches each result back to its source row on
> n8n's built-in `row_number` from the read step — **no `id` column is added**.
> This is robust against duplicate or blank `company_name`.

---

## Setup

### Prerequisites
- Node.js 18+ (the primary path runs n8n via `npx`), **or** Docker Desktop (the
  alternate path).
- A Google account (for the Sheet + OAuth).
- A **Google Gemini API key** ([AI Studio](https://aistudio.google.com/)).
  Use a **dedicated key for this project** so its quota is isolated.

### 1. Start n8n

**Primary — local Node:**
```bash
npx n8n
```
Then open http://localhost:5678.

**Alternate — Docker:** the included `docker-compose.yml.txt` (rename to
`docker-compose.yml`) bind-mounts `./.n8n` so both paths share one data dir:
```bash
docker compose up -d
```

### 2. Add the Gemini API key as a credential (never in the workflow file)
In n8n: **Credentials → New → Header Auth**
- **Name:** `Gemini API key` (referenced by name; the key itself stays in n8n's
  encrypted store)
- **Header Name:** `x-goog-api-key`
- **Header Value:** *your Gemini API key*

### 3. Connect Google Sheets
Add a **Google Sheets OAuth2** credential and authorise your account.

### 4. Import the workflow
**Workflows → Import from File →** `leadforge-workflow.json`. Then wire up the
per-node config that can't be shipped in a file:
- **Read Leads** and **Write Back to Sheet**: set your Google Sheet (replace
  `REPLACE_WITH_YOUR_GOOGLE_SHEET_ID`) and select the Google Sheets credential.
- **Enrich via Gemini**: select the `Gemini API key` Header Auth credential.

### 5. Set the model and ICP
Open the **Config** node — the single source of truth for both swappable values:
- `GEMINI_MODEL` — the model string (see the note below).
- `ICP` — the scoring profile. Swap this block to retarget the tool.

### 6. Run
Click **Execute Workflow** and watch the sheet fill in.

---

## ⚠️ Model string — `GEMINI_MODEL`

The model is set in **one place only**: the `GEMINI_MODEL` field of the **Config**
node. Every node that needs it references that field — nothing else hardcodes a
model string.

The shipped value is **`gemini-2.5-flash`** (free-tier on the author's key). If
enrichment calls ever start returning 404 / "model not found" — Google does
rotate model availability over time — open the Config node and change this one
field to a current free-tier model (check the model list in
[AI Studio](https://aistudio.google.com/)). No other node needs editing.

## Quota & pacing

The Gemini free tier is **rate-limited per minute (RPM)** and has a **daily
request cap**. **The exact RPM/RPD numbers are not hardcoded here** — they vary
by model and by project and change over time. **Read your actual limits in the
[AI Studio](https://aistudio.google.com/) quota page** for your key.

The workflow paces itself rather than relying on a specific number:
- A **Wait node (~6s)** sits after each successful Gemini call. If your RPM is
  higher or lower, tune this one value.
- **Split In Batches (size 1)** processes one lead at a time for orderly runs
  and clean interaction with the Wait node — it is **not** a quota control.
- A quota error (429) on any lead routes that lead to `needs_manual_review`
  (`llm_call_failed`) and the run continues; nothing already written is lost.
- The **skip-already-enriched** gate means re-running never re-scores finished
  rows, so you don't burn quota on repeat runs.

Size your demo lead list to fit comfortably under the daily cap.

## Security

- The API key is entered through the n8n UI into its **credential store** and
  referenced by name. It is **never** in `leadforge-workflow.json`, `.env`, or
  any committed file.
- `.n8n/` (which holds the encrypted credential store) and `.env` are
  git-ignored.
- Fetched website text is treated as **untrusted data** in the prompt (delimited
  and explicitly marked "not instructions") to blunt prompt-injection from a
  scraped page.
- Runs locally only — not deployed, so the quota can't be drained by strangers.

## Tuning

In the **Strip HTML + Cap Size** node:
- `MIN_CLEAN_CHARS` (default 250) — gate 3 threshold. Raise it if junk pages
  slip through; lower it if legitimate thin sites get bounced.
- `MAX_CLEAN_CHARS` (default 8000) — how much clean text is sent to the LLM.

## Swapping the ICP — worked example

The ICP is a single text block in the **Config** node, injected into the
enrichment prompt. Nothing about the pipeline is tied to the demo's AI-consultancy
target — swap that block and the *same* leads get re-scored against a different
buyer. To show this on paper without a second live run, here is the demo's
recruitment-agency alternative:

> **IDEAL CUSTOMER PROFILE (recruitment agency — permanent & contract placement):**
> A good lead is a company likely to hire in volume: a growing headcount, multiple
> open roles, seasonal or shift-based workforces, high staff turnover, or physical
> operations that scale with labour (logistics, trades, field services, security).
> Professional-services firms that bill by headcount also fit. Score 8–10 = strong
> hiring signal, 4–7 = possible, 1–3 = poor fit.

The score a lead gets is *ICP-relative*, not absolute — the same company can be a
strong fit for one buyer and a weak fit for another. Illustrative (hypothetical)
scores for four of the demo leads under each ICP:

| Lead | AI-consultancy ICP | Recruitment-agency ICP | Why they differ |
|---|---|---|---|
| Direct Couriers | 7 — manual dispatch/ops ripe for automation | 8 — large shift-based driver workforce | Both score well, for opposite reasons |
| Programmed Maintenance | 8 — field-service scheduling, automatable | 9 — staffing/labour-hire is their core business | Recruitment ICP rates it higher |
| Little Real Estate | 6 — CRM/lead workflows to automate | 7 — high-turnover agent hiring | Comparable under both |
| Ronstan | 5 — niche manufacturer, leaner ops | 3 — small, stable specialist workforce | AI ICP rates it higher |

These numbers are illustrative of *how the axis changes*, not measured outputs —
run the workflow with each ICP block to get real scores. The point: **one field
changes, and the pipeline retargets.**

## Limitations & production hardening (Phase 2)

These are left out of v1 **by choice**, to keep the demo small and honest, not
because they were overlooked. A production deployment would add them:

- **Semantic relevance check.** Gate 3 is a *length* heuristic — it confirms
  enough clean text came back, not that the text is *about the company*. A page of
  generic boilerplate can still pass. A Phase-2 relevance check (keyword/entity
  match, or a cheap first-pass "is this a real company page?" classifier) would
  close that gap.
- **Agentic decision layer** (see Roadmap below).
- **Run metrics / observability.** No per-run counts, timings, or a summary node.
  A production build would emit enriched-vs-review tallies and durations.
- **Pagination / scale.** Reads the sheet in one pass; fine for a demo list, but a
  multi-thousand-row sheet needs batched reads and resumable runs.
- **Structured logging.** Failures route to `review_reason` in the sheet, but there
  is no separate run log for debugging beyond n8n's execution view.
- **Automated tests.** Gate behaviour is verified manually against the demo CSV
  (see the outcomes table above), not by an automated suite.
- **ICP versioning.** The ICP lives in one editable field; there is no history of
  which ICP scored which row. A production build would stamp the ICP version onto
  each enriched row.

## Roadmap (Phase 2 — documented, not built)

An **agentic decision layer**: instead of the fixed linear path, an LLM decides
per lead which enrichment steps to run (e.g. website present → scrape it; only
an email → try a domain lookup instead). This is what elevates it from
"automation" to "agentic automation." The current per-lead branching is the seed
of it.

## Out of scope

Paid APIs / data providers, CRM integrations beyond Google Sheets, public
deployment.
