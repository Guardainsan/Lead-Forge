# ARCHITECTURE-ESSENTIALS.md — scan this every prompt

Quick outline of the lead-enrichment pipeline. Full detail in ARCHITECTURE.md;
what/who in PROJECT.md. Read THIS one each turn; open ARCHITECTURE.md only when
you need node-level or data-model detail.

## One line
n8n (self-hosted) reads leads from a Google Sheet, fetches each company's
website, strips it to clean text, and — **only if enough text came back** — has
Gemini score it against a configurable ICP, then writes the result back to the
sheet. General-purpose tool; demoed with an AI-consultancy ICP.

## Stack (fixed)
- **n8n**, run locally via **Node** (`npx n8n`) at `localhost:5678`.
  Docker compose is kept as the documented **alternate** path.
- **Google Gemini**, free tier. Model string lives in ONE place: the
  `GEMINI_MODEL` field of the Config node (currently `gemini-2.5-flash`).
- **Google Sheets** = input AND output (OAuth-connected)
- **HTTP Request** node = web fetch *and* the Gemini call (needed for
  `responseSchema`)
- Local only, API key in n8n's credential store, NOT deployed. No GPU.

## Data shapes
- **Input row:** `company_name` (req), `website` (opt), `email` (opt)
- **Output cols:** `summary`, `industry`, `fit_score` (1–10),
  `recommended_action`, `enriched_at`, `status` (`enriched` |
  `needs_manual_review`), `review_reason`
- **LLM returns JSON via native structured output** (`responseMimeType` +
  `responseSchema`): `{summary, industry, fit_score, recommended_action}`
- **Row identity:** n8n's built-in `row_number` from the read step. It is the
  write-back matching key. No `id` column is added to the sheet.
- **ICP** = one editable text block in the Config node, injected into the
  prompt — the thing you swap to retarget the tool.

## Workflow (v1, linear)
```
Trigger → Config (ICP + GEMINI_MODEL) → Read Sheet
  → skip rows already enriched
  → loop per lead
      → has company_name?  no ──────────────────────┐
      → has website?  no ──────────────────────────┤
         yes → fetch site (UA header, continue-on-fail)
              → error ────────────────────────────┤
              → strip HTML → clean text            │
                  → enough real text?  no ────────┤  → needs_manual_review
                     yes → Gemini (structured JSON)│
                          → shape fields ──────────┴─→ write back (match row_number)
                             → wait ~6s → next lead
```

## The grounding gates (non-negotiable — this is the project's core claim)
The LLM is not called for a lead we have no real data on. **Four** gates sit in
front of it, all deterministic, all before any token is spent:

0. **No company name** → `needs_manual_review` (input validation; the one
   required field).
1. **No website** → `needs_manual_review`.
2. **Fetch failed** (dead host, timeout, 403, non-2xx) → `needs_manual_review`.
3. **Empty-200 / thin content** → `needs_manual_review`.
   A 200 response is *not* evidence. JS-only shells, bot-walls, parked domains
   and cookie interstitials all return 200 with no company information. After
   HTML is stripped, clean text below the minimum length fails the gate.
   **This gate does the load-bearing hallucination-risk reduction.** Without it
   the model is handed an empty string and invents a company. Gates 0–2 are
   hygiene; gate 3 is the point.
   **Honest caveat:** gate 3 checks *length*, not *relevance* — boilerplate can
   still pass. Claim is "substantially reduces hallucination risk, especially for
   empty/unfetchable sites," NOT "never fabricates." Semantic relevance check =
   Phase 2.

Every fallback row records *which* gate it hit in `review_reason`, so a
reviewer can see the behaviour working rather than take it on trust.

## Other rules
- **Structured output, not prompt-please-JSON.** `responseSchema` is set on the
  API call. There is no parse-retry loop — the schema makes it unnecessary.
- **Graceful failure.** Any single lead failing never kills the run; enriched
  rows already written stay written.
- **Quota.** Free tier is RPM-limited. A Wait node (~6s) after each Gemini call
  paces the run. Read your actual RPM/RPD in AI Studio — do not assume numbers.
- **Split In Batches (size 1)** is for orderly one-lead-at-a-time processing and
  clean interaction with the Wait node. It is **not** a quota control.
- **Configurable ICP**, not hardcoded to AI consultancy.
- **Key never in the workflow JSON** — n8n credential store, referenced by name.

## Scope
- **v1 (build now):** the flow above, 4 enrichment fields, 4 grounding gates,
  one example ICP.
- **Phase 2 (documented, NOT now):** agentic layer — an LLM decides per lead
  which enrichment steps to run.
- **Out:** paid APIs, CRMs beyond Sheets, any hosting/deploy.

## Why this project exists
Portfolio project 2 — demonstrates **agentic automation + workflow tooling**,
deliberately different from the RAG contract auditor (project 1). Business-value
framing: turns hours of manual lead research into an automated, AI-scored,
reusable pipeline.
