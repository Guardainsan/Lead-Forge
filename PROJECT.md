# PROJECT.md — Automated Lead Enrichment Pipeline

## What this is
An automated lead-enrichment pipeline built in **n8n** (self-hosted via Docker).
It takes a list of raw leads from a **Google Sheet** (company name / website /
email), automatically researches each one, uses an **LLM (Google Gemini)** to
summarise and score it, recommends an action, and writes the enriched result
back to the same sheet. A messy input list becomes a clean, AI-scored,
sortable, actionable lead list.

It is **general-purpose**: the scoring target — the Ideal Customer Profile
(ICP) — is a **configurable input**, so the same pipeline scores leads for any
business. It is **demoed** with one example ICP — an **AI consultancy** (a
company that sells AI/automation services) — but nothing is hardcoded to that
industry. Swap one ICP block and it scores leads for a different sector.

## Who it's for
- **Primary (portfolio) audience:** AI/ML and automation recruiters / hiring
  managers in Melbourne evaluating this as project 2 of a portfolio. It exists
  to demonstrate **agentic automation and workflow-tool skills** — deliberately
  DIFFERENT from the author's RAG contract-auditor project, so the portfolio
  reads as range (RAG + agentic automation + game AI), not one trick twice.
- **Notional end user:** anyone doing lead research — a sales/BD person at a
  consultancy or agency who would otherwise spend hours manually researching and
  qualifying a list of leads. One-line value: *turns a manual, hours-long
  lead-research task into an automated, AI-scored pipeline.*

## What makes it credible (not a toy)
- **Structured LLM output** — the enrichment step uses Gemini's native
  structured output (`responseSchema`), so results come back as guaranteed JSON,
  reliably writable back to a sheet — no fragile "please return JSON" prompting
  or parse-retry loops.
- **Grounded scoring — substantially reduced hallucination risk** — the LLM only
  ever scores a lead we actually fetched text for. Four deterministic gates run
  before the model is called: no company name, no website, failed fetch, and —
  the important one — a **thin-content check** that catches pages returning HTTP
  200 with no usable information (JS-only shells, bot-walls, parked domains). A
  "successful" fetch is not treated as evidence until it's shown to carry text.
  This mirrors the author's auditor project: no evidence → don't assert, mark
  "needs manual review" instead.
  **Stated honestly:** gate 3 is a *length* heuristic, not a semantic relevance
  check — a page of generic boilerplate can still pass it. The claim is
  "substantially reduces hallucination risk, especially for empty or unfetchable
  sites," not "never fabricates." A semantic relevance check is a Phase-2 item.
- **Graceful failure** — any single lead failing (missing site, dead host, thin
  content, quota error) branches to a "needs manual review" fallback that
  records *why*, and never crashes the run; rows already enriched stay written.
  Robustness is a first-class feature, not an afterthought.
- **Configurable, not hardcoded** — swap one ICP block and it scores for a
  different industry. Reusability is the point.
- **Self-hosted infrastructure** — n8n runs locally on the author's machine
  (under Node; Docker compose provided as an alternate path), demonstrating the
  ability to stand up and run a real service (an ops signal), and giving
  unlimited workflow executions vs. n8n cloud's capped free tier. The Gemini API
  key lives in n8n's credential store, never in the workflow file.

## Scope
**In scope (v1 — linear pipeline):**
- Google Sheet input → per-lead web research → HTML→text → LLM enrichment →
  write back to the correct row (matched on n8n's built-in `row_number`)
- Four enrichment fields per lead: `summary`, `industry`, `fit_score` (1–10),
  `recommended_action` (plus `status`, `enriched_at`, `review_reason`)
- Three grounding gates before the LLM (no website / failed fetch / thin
  content), each routing to a "needs manual review" fallback that records why
- Skip-already-enriched gate so re-runs don't re-score finished rows
- One example ICP (AI consultancy), documented and configurable
- Quota-aware pacing via a Wait node (free tier is RPM-limited; exact limits are
  read from AI Studio, not assumed — the demo runs on a small lead list)

**Phase 2 (planned, NOT blocking v1):**
- **Agentic decision layer** — instead of a fixed linear sequence, an LLM decides
  per lead which enrichment steps to run (e.g. website present → scrape it; only
  an email → try a domain lookup instead). This is what elevates it from
  "automation" to "agentic automation."

**Out of scope:**
- Paid APIs or paid data providers (free tooling only)
- CRM integrations beyond Google Sheets
- Public deployment / hosting (runs locally, shown via demo video + a repo /
  exported workflow JSON)

## Constraints
- **Free tooling only.** Gemini free tier — **RPM-limited** with a daily request
  cap. Exact RPM/RPD numbers are read from Google AI Studio for this project's
  dedicated key, not assumed; the workflow paces itself with a Wait node and the
  demo list is small. The model string (`GEMINI_MODEL`, currently
  `gemini-2.5-flash`) is a single swappable placeholder.
- **Self-hosted n8n** running locally under Node (`localhost:5678`); a Docker
  compose file is provided as an alternate path. No GPU required — n8n
  orchestrates API calls; the LLM runs on Google's servers.
- **Google Sheets** as both input and output (via n8n's Google Sheets node,
  OAuth-connected).
- **API key never committed** — entered via the n8n UI into its credential
  store and referenced by name; `.env` and `.n8n/` are git-ignored.

## Success criteria
A reviewer can: open the demo, see a Google Sheet of raw leads, trigger the n8n
workflow, and watch the sheet fill in with a summary, industry, 1–10 fit score
(scored against a stated ICP), and a recommended action per lead — with
missing-data rows handled gracefully rather than failing the run. The workflow
is visual (screenshots / video-friendly), the ICP is shown to be swappable, and
the whole thing runs on free tooling.
