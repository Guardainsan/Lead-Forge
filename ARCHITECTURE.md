# ARCHITECTURE.md — Automated Lead Enrichment Pipeline

> Full architecture, stack, data models, node-by-node workflow, and design
> decisions. For a quick scan every prompt, read ARCHITECTURE-ESSENTIALS.md
> instead; this file is the deep reference.

## 1. Stack

| Layer | Choice | Why |
|---|---|---|
| Orchestration | **n8n**, run locally via **Node** (`npx n8n`), at `localhost:5678`. Docker compose kept as the documented alternate path. | Visual workflow tool employers use. Running it locally under Node is the author's day-to-day setup; Docker is provided for anyone who prefers a container. |
| LLM | **Google Gemini** (free tier). Model string is a single placeholder, `GEMINI_MODEL`, in the Config node — currently `gemini-2.5-flash`. | Free-tier. Isolated to one field so it can be swapped in one place if Google rotates model availability. Called via the **HTTP Request** node (the generic-node path is required to send `responseSchema`). |
| Input / Output | **Google Sheets** (one sheet, enriched columns written back) | Simplest, most visual, zero extra tool; the "watch the sheet fill in" demo is the strongest visual. OAuth-connected in n8n. |
| Web fetch | n8n **HTTP Request** node against each lead's public website | Free, no data-provider API key; turns "just a name" into raw HTML to score. |
| Host | Local machine (Windows). Key lives in n8n's credential store. | Not deployed — key stays local, quota not drainable by strangers. |

**No GPU needed** — n8n is I/O orchestration; the model runs on Google's servers.

### Model & quota — read, don't assume
- `GEMINI_MODEL` is set **once** in the Config node and referenced by every node
  that needs it. Currently `gemini-2.5-flash`, confirmed free-tier on this
  project's key. If Google ever rotates it out, change this one field (and the
  README note) — nothing else references a model string.
- The API key is entered through the n8n UI into the **credential store** and
  referenced by name. It is **never** written into the workflow JSON, `.env`, or
  any committed file. This project uses a **dedicated key** with its own quota,
  separate from the author's other projects.
- The free tier is **RPM-limited** (and has a daily request cap). Exact RPM/RPD
  numbers are **not asserted here** — they change per model and per project and
  must be read from **Google AI Studio** for the active key. The workflow is
  built to *pace itself* (Wait node) rather than to depend on a specific number.

## 2. Data models

### Input row (Google Sheet, one lead per row)
```
company_name : string   (required)
website      : string   (optional — may be blank)
email        : string   (optional — may be blank)
```
The demo sheet deliberately includes rows with a **missing website** and rows
whose site returns **no usable text**, because those rows exercise the grounding
gates (§3) and justify the Phase-2 agentic branch (different path per lead).

Shipped as `leadforge-leads.csv` — 8 leads chosen so **every gate fires**: 4 that
enrich cleanly, 1 blank website (gate 1), 1 verified 403 bot-wall (gate 2), and 2
distinct flavours of thin content (gate 3) — a parked-style page returning 127
clean chars and a JS-only SPA shell returning **0**. The expected outcomes were
measured with the workflow's own User-Agent, not assumed; the measurements and a
drift warning are in the README.

### Row identity — the write-back matching key
n8n's Google Sheets **read** step exposes a built-in `row_number` for every row.
That value is carried through the whole flow and used as the **matching column**
on write-back, so each enriched result lands on exactly the row it came from —
even if `company_name` is duplicated or blank. **No `id` column is added to the
sheet**; `row_number` is sufficient and requires no change to the input data.

### Enriched output (written back to the same row)
```
summary            : string          (1–2 sentence company summary)
industry           : string          (classified sector)
fit_score          : integer 1–10    (fit against the configured ICP)
recommended_action : string          (e.g. "High fit — reach out directly" /
                                       "Low fit — deprioritize")
enriched_at        : ISO timestamp   (set on BOTH paths)
status             : "enriched" | "needs_manual_review"
review_reason      : string          (why a row went to manual review:
                                       "no_company_name" | "no_website" |
                                       "fetch_failed" | "thin_content" |
                                       "llm_call_failed"; empty when enriched)
```
`status` and `enriched_at` are written on the **success path too**, not only on
the fallback — the enriched branch sets `status = enriched`. `review_reason`
records which grounding gate fired, so the behaviour is visible in the sheet.

### LLM enrichment contract (native structured output)
The Gemini call sets `responseMimeType: "application/json"` and a
`responseSchema`, so the model returns exactly this object — no prose, no
markdown fences, no parsing guesswork:
```json
{
  "summary": "string",
  "industry": "string",
  "fit_score": 0,
  "recommended_action": "string"
}
```
Because the schema is enforced by the API, there is **no parse-retry loop**. If
a call genuinely fails (network / quota error), that single lead routes to
`needs_manual_review` like any other failure; the run continues.

### ICP (configurable — the one thing you swap per use case)
Stored as a single text block in the **Config** node (alongside `GEMINI_MODEL`),
injected into the enrichment prompt. Demo value:
```
IDEAL CUSTOMER PROFILE (AI consultancy):
A good lead is a mid-sized company (roughly 20–500 staff) that shows signals it
would buy AI/automation services — e.g. manual/repetitive operations, a growing
tech or data team, recent digital-transformation or hiring signals, or an
industry ripe for automation. Score 8–10 = strong fit, 4–7 = possible,
1–3 = poor fit.
```
Swap this block → the same pipeline scores leads for any other business.

## 3. Grounding gates — substantially reduced hallucination risk

**The LLM is not called for a lead we have no real data on.** This mirrors the
auditor project's rule (no evidence → don't assert). Four deterministic gates sit
in front of Gemini; each writes its reason to `review_reason` and skips the model
entirely (no wasted quota, no invented company):

0. **No company name** — the row has a blank `company_name` (the one required
   field). Input validation: without it there is nothing to enrich, so the row
   never reaches the fetch. → `needs_manual_review`, reason `no_company_name`.
1. **No website** — the row has a blank `website`. → `needs_manual_review`,
   reason `no_website`.
2. **Fetch failed** — the HTTP Request errors or returns a non-2xx (dead host,
   timeout, 403 bot-wall, 5xx). Handled via **Continue-On-Fail** so the run does
   not halt; the error item routes to fallback. → reason `fetch_failed`.
3. **Empty-200 / thin content** — *the important one.* A 200 status is **not**
   evidence. JS-only single-page apps, bot-walls, parked domains, and
   cookie-consent interstitials routinely return 200 with no company text. After
   the HTML is stripped to plain text (§4, node 6), if the clean text is below a
   **minimum length threshold**, the lead fails the gate. → reason
   `thin_content`.

Gate 3 is what most reduces hallucination risk: without it, the model receives an
empty or junk string on a "successful" fetch and hallucinates a plausible
company. Gates 0–2 are hygiene; **gate 3 does the load-bearing work.**

**Honest limitation.** Gate 3 is a **length heuristic, not a semantic relevance
check.** It verifies that *enough clean text* returned, not that the text is
*about this company*. A page carrying a few hundred characters of generic
boilerplate passes the gate, and the model may over-read it. So the project
claims **"substantially reduced hallucination risk, especially for empty or
unfetchable sites"** — not "never fabricates." A **semantic relevance check**
(does the fetched text actually mention the company or contain business signals?
e.g. a cheap keyword/entity check, or a first-pass LLM "is this a real company
page?" classifier) is a **Phase-2** improvement, deliberately out of v1 scope.

## 4. Workflow — node by node (v1, linear)

```
[1 Trigger]
  → [2 Config: ICP + GEMINI_MODEL]
  → [3 Read Sheet (exposes row_number)]
  → [4 Skip already-enriched]  (status not empty → drop row)
  → [5 Loop / Split In Batches, size 1]
      → [5a Has company_name?] --no--> [FB Fallback builder] → [12 Write back]
      → [6 Has website?] --no--> [FB Fallback builder] → [12 Write back]
                         --yes-> [7 HTTP fetch (UA header, Continue-On-Fail)]
             --error/ non-2xx-------------------------> [FB Fallback builder] → [12]
                         --ok--> [8 Strip HTML → clean text]
                                   → [9 Cap size (truncate to N chars)]
                                     → [10 Enough real text?]
                                        --no--> [FB Fallback builder] → [12]
                                        --yes-> [11 Gemini enrich (structured JSON)]
                                                  → [11b Shape fields, status=enriched]
                                                     → [12 Write back (match row_number)]
                                                        → [13 Wait ~6s] → next lead
```

1. **Trigger** — Manual Trigger for the demo (click "Execute"). Swappable for a
   Sheets "row added" trigger or Webhook later; manual is the most
   demo-controllable.
2. **Config (Set node)** — holds the `ICP` text block and `GEMINI_MODEL`. The
   single source of truth for both swappable values. Nothing else hardcodes a
   model string or ICP.
3. **Read Sheet** — Google Sheets node reads all lead rows and exposes
   `row_number` per row (the write-back key).
4. **Skip already-enriched** — an IF/Filter that drops rows whose `status` is
   already set. Re-running the workflow does not re-score finished rows or waste
   quota.
5. **Loop / Split In Batches (size 1)** — process one lead at a time for orderly
   per-lead handling and clean interaction with the Wait node (node 13). **This
   is not a quota control** — the Wait node is.
5a. **Branch: has company_name?** — IF node (input validation, gate 0). Blank
   `company_name` → fallback (`no_company_name`); present → continue. `company_name`
   is the one required field, so an empty one has nothing to enrich.
6. **Branch: has website?** — IF node. Blank `website` → fallback (`no_website`);
   present → fetch path. (Seed of the Phase-2 agentic decision layer.)
7. **HTTP fetch website** — HTTP Request node, homepage. Sends a real
   **User-Agent** header (default n8n UA is widely 403'd). **Continue-On-Fail is
   ON**; on error / non-2xx the item routes to fallback (`fetch_failed`) instead
   of halting the workflow.
8. **Strip HTML → clean text** — Code node (or HTML Extract): removes
   `<script>`, `<style>`, nav/markup, collapses whitespace. Raw HTML is never
   sent to the model — it wastes tokens and can blow the per-minute token limit.
9. **Cap size** — truncate the clean text to a fixed character budget before the
   LLM, bounding token usage per call.
10. **Enough real text?** — IF node: clean-text length ≥ threshold? No → fallback
    (`thin_content`, gate 3). Yes → enrich.
11. **LLM enrich** — HTTP Request to Gemini using `GEMINI_MODEL`, the injected
    ICP, and the capped text, with `responseMimeType: application/json` +
    `responseSchema`. Returns the four-field object directly.
    - **11b Shape fields** — Set node maps the JSON into the output columns and
      sets `status = enriched`, `enriched_at = now`, `review_reason = ""`.
    - **FB Fallback builder** — Set node used by every gate: `status =
      needs_manual_review`, `enriched_at = now`, `review_reason =` the gate that
      fired. Never calls the LLM.
12. **Write back** — Google Sheets **Update**, matching on `row_number`, writes
    the enriched/fallback columns onto the correct row.
13. **Wait ~6s** — paces requests under the free-tier RPM limit before the next
    lead. Tune to your AI Studio RPM.

### Failure handling (first-class, per PROJECT.md)
- Missing website → `needs_manual_review` (`no_website`) before any LLM call.
- HTTP fetch fails / non-2xx → `needs_manual_review` (`fetch_failed`), run
  continues (Continue-On-Fail).
- Thin / empty-200 content → `needs_manual_review` (`thin_content`) — gate 3.
- LLM call errors (network / quota) → that lead → `needs_manual_review`; the run
  does not crash and already-written rows stay written.
- Daily quota exhausted → later leads simply fail their LLM call and fall back;
  the sheet keeps everything already enriched. Demo list sized to the quota.

## 5. Key design decisions (chosen vs. rejected, and why)

- **Google Sheet in/out, not Airtable/CRM** — simplest, most visual, zero new
  tool; the sheet-filling-in demo is the strongest recruiter-facing visual.
- **`row_number` as the write-back key, not a new `id` column** — n8n exposes it
  for free on read; adding an `id` column would mean editing the input data for
  no benefit. Robust against duplicate/blank `company_name`.
- **Strip HTML before the LLM** — raw pages are 100s of KB of nav/script/CSS;
  sending them wastes tokens and risks the per-minute token cap. Clean text only.
- **Gate 3 (thin-content) as a first-class grounding rule** — a 200 is not
  evidence. This does the load-bearing hallucination-risk reduction, not an edge
  case. Accepted trade-off: it is a length heuristic, so a boilerplate page can
  still pass; the semantic check is deferred to Phase 2 rather than faked in v1.
- **Native structured output, no parse-retry loop** — `responseSchema` makes
  malformed JSON a non-problem; the retry machinery would be dead complexity.
- **HTTP Request node for Gemini, not a prebuilt node** — needed to pass
  `responseSchema`; keeps the model call explicit and portable.
- **Wait node for pacing, Split-In-Batches for order** — separating these two
  concerns; the earlier spec wrongly credited batching with quota control.
- **`recommended_action` field, not just a score** — tells the user what to
  *do*; that's the business value student projects usually lack.
- **Deterministic fallback before the LLM, not "let the model handle it"** — a
  lead with no data must never be scored; branch it out first.
- **Model string isolated to one field** — one-line swap if the model ever
  rotates.
- **Linear v1, agentic layer as Phase 2** — keeps this project deliberately
  smaller/faster than the auditor. The Phase-2 agent makes an LLM choose the
  enrichment path per lead; documented but not built in v1.

## 6. Constraints & open decisions
- **Binding:** free tooling only; Gemini free-tier (RPM-limited, exact numbers
  from AI Studio); local-only, key in the credential store; Google Sheets as the
  store; n8n under Node (Docker as alternate).
- **Open (a builder must not invent these):** the exact `GEMINI_MODEL` value is
  owner-supplied (currently `gemini-2.5-flash`);
  how many demo leads (sized to the quota); the clean-text length threshold and
  size cap (tune on real pages); whether the final demo uses a manual or
  sheet-row trigger; Phase-2 agent design.
