---
name: job-search
description: On-demand job-search pass for Nikita. Pulls jobs from a mock dataset (jobmcp) or live providers (realjobmcp / tavilysearch) selectable via the `backend=` modifier, filters against the target profile in MEMORY.md, gathers company context from companymcp, builds a per-job briefing with tiered LLM escalation (Gemini → Haiku → Sonnet → Opus), emails it via gmail-mcp, posts a Slack heads-up, and on a "yes" reply submits a mock application (mock backend only) via jobmcp. Trigger phrase: "run job-search pass" (with optional `query=` / `limit=` / `backend=mock|real` / `mock_fallback=on|off` / `dry-run=true` modifiers).
metadata:
  { "openclaw": { "emoji": "💼", "requires": { "config": [] } } }
---

# job-search

On-demand pipeline that turns raw job listings into a decision-ready briefing for Nikita and queues the apply step on his approval.

## When to invoke

- User says **"run job-search pass"** (with or without modifiers).
- Optional modifiers in the same message:
  - `query="senior data engineer remote"` — overrides the default search query (default is built from `MEMORY.md → TARGET_ROLES`).
  - `limit=N` — max number of jobs to brief in this pass (default 5).
  - `backend=mock|real` — choose the job-listing source (default `mock`).
    - `mock` → `jobmcp` (curated mock dataset; supports the full apply-mock path).
    - `real` → `realjobmcp` (live providers via tavilysearch: tavily/remotive/etc.). The mock-application step is unavailable on this backend; the reply-handler instead asks Nikita to apply manually via `application_url`.
  - `mock_fallback=on|off` — when `backend=real` and 0 listings survive hard-filtering, transparently re-run Step 1 against `jobmcp` so the rest of the pipeline (company context → briefing → email → Slack) still has something to deliver. Default `on`. Each fallback briefing is tagged `via_fallback=mock` in `memory/YYYY-MM-DD.md` so it's easy to tell apart from real-source briefings.
  - `dry-run=true` — produce briefings but skip Gmail send + Slack post (preview only — print the briefings to chat).

Do **not** auto-trigger this skill. It's user-initiated.

## Required context (read once at start of pass)

1. `MEMORY.md` — Nikita's skill profile, target-role spec (`TARGET_ROLES`, `ROLES_TO_AVOID`, `GEO_HARD_FILTERS`, `SENIORITY`, `COMP`, `LANGUAGE`, `EXTRA_PREFERENCES`, `DEAL_BREAKERS`), and the model-tiering policy.
2. `cv.txt` — full CV text. Use as ground truth for skill matching, not the abbreviated profile in MEMORY.md.
3. **Persistent processed-job set in Valkey** — the dedupe source of truth. Read it at start of pass via `exec`:
   `node /home/node/.openclaw/workspace/scripts/processed.js list` → one job-id per line.
   Backed by Valkey set `jobsearch:processed`; persists across days and survives `memory/*.md` edits. Do **not** parse markdown for this purpose.

## Connected MCP servers (already wired)

| Server   | Tools                                                                                 |
| -------- | ------------------------------------------------------------------------------------- |
| `jobmcp` | `search_jobs`, `get_job`, `list_companies`, `submit_mock_application` (mock backend) |
| `realjobmcp` | `search_job` (singular), `list_job_sources` (live backend, tavilysearch)            |
| `company`| `company_profile`, `recent_news`, `linkedin_company_lookup`, `linkedin_lookup`, `cached_company_results` |
| `slack`  | read tools + `conversations_add_message` (channel `C0B0K3X8VND` = `#job-search`)      |
| `gmail`  | `send_email(subject, body_markdown, to=None, body_html=None)`, `whoami`                |

Tools are namespaced by server name in openclaw — e.g. `jobmcp__search_jobs`, `company__company_profile`, `slack__conversations_add_message`, `gmail__send_email`.

## Pipeline

### Step 0 — bootstrap

Read `MEMORY.md`, `cv.txt`, today's `memory/YYYY-MM-DD.md` (create if missing). Parse `TARGET_ROLES`, `GEO_HARD_FILTERS`, `DEAL_BREAKERS`, `COMP`, `SENIORITY`. Pull the persistent processed-set: `processed_ids = exec("node /home/node/.openclaw/workspace/scripts/processed.js list").split("\n")`. Confirm `gmail__whoami` returns the Resend backend (one call, swallow on error and log a warning to chat).

### Step 1 — search

#### 1a. Build the search query

If the user passed an explicit `query=` modifier, use it verbatim. Otherwise derive a candidate-aware query from `cv.txt` + `MEMORY.md`:

1. From `cv.txt`, extract the candidate's core stack: pull skills/tools that appear in the most recent 2-3 roles (e.g. languages, frameworks, data tools). Exclude generic "soft" skills.
2. Combine with `MEMORY.md → TARGET_ROLES` (the role title) and `MEMORY.md → SENIORITY.min` (the seniority).
3. Keep the query short (≤ 12 words) and provider-friendly: real job boards weight earlier terms higher and choke on long boolean expressions.
4. Skew toward concrete tech keywords over abstractions: `"senior python data engineer airflow dbt clickhouse remote"` beats `"senior backend engineer"`.

Pseudocode:
```
core_stack = cv_top_skills(top_n=4)                       # e.g. ["Python", "Airflow", "dbt", "ClickHouse"]
target = MEMORY.TARGET_ROLES[0]                           # e.g. "Senior Data Engineer"
mode   = "remote" if MEMORY.GEO_HARD_FILTERS.remote else ""
query  = f"{target} {' '.join(core_stack)} {mode}".strip()
```

Narrate the derived query so the run is reproducible: `[step 1a] derived query: "<...>"`.

#### 1b. Call the search tool

Branch on the `backend` modifier (default `mock`):

- **`backend=mock`** → call `jobmcp__search_jobs(query, ...)`. Returns curated mock listings with stable string ids. Cap raw results at 30.
- **`backend=real`** → call `realjobmcp__search_job(query=..., limit=30, work_mode="remote"|..., sources=[...optional])`. Tool name is **singular** (`search_job`, not `search_jobs`). The result includes a `jobs` array; each job has `source_job_id`, `source`, `url`, `application_url`, `title`, `company`, `location`, `work_mode`, `description`, `tags`, etc. Use `{source}:{source_job_id}` as the canonical id (e.g. `remotive:2090000`); if `source_job_id` is missing, fall back to a sha1 of `url` truncated to 12 hex chars.

Drop anything whose canonical id appears in `processed_ids` (the persistent set loaded in Step 0). The same `jobsearch:processed` Valkey set is shared between both backends — pre-namespaced ids (`remotive:...`, `tavily:...`) keep them from colliding with mock ids.

#### 1c. Mock fallback (only when `backend=real`)

After Step 2 (hard-filter), if the `backend=real` search produced **0 survivors** AND `mock_fallback` is not explicitly `off`:

1. Narrate clearly: `[step 1c] real backend returned 0 survivors after hard-filter — falling back to jobmcp (mock) so the rest of the pipeline can still deliver`.
2. Re-run Step 1b against `jobmcp__search_jobs` with the same derived query.
3. Re-run Step 2 (hard-filter) over the mock result set.
4. Set `via_fallback = "mock"` on every job from this round; it will be recorded in the briefing entry in `memory/YYYY-MM-DD.md` so reviewers can spot fallback briefings at a glance.
5. The reply-handler's "yes" path still uses the **original** `backend` setting from this pass (so a fallback-mock briefing also goes through `jobmcp__submit_mock_application` because its job is a jobmcp job with a real string id). This keeps the apply-mock affordance available even for fallback runs.

If the real search returned ≥1 survivor, Step 1c is skipped entirely.

### Step 2 — hard-filter (cheap, no LLM)

Reject deterministically (no LLM call):
- Country/city not in `GEO_HARD_FILTERS.allowed_*` AND `remote_ok` is false in the listing.
- `visa_sponsorship_required: true` and listing explicitly says "no sponsorship".
- Any `DEAL_BREAKERS` keyword match in title or description (substring, case-insensitive).
- Comp listed and below `COMP.min_base_usd`.
- Seniority outside `SENIORITY.min`-`SENIORITY.max`.

If >10 candidates remain after deterministic filters, escalate to **`cheap`** (Haiku 4.5) with a single batched prompt: "For each job below, output JSON `{id, keep: bool, reason}`. Keep iff it plausibly matches `<target profile JSON>`." Drop the `keep: false` ones.

### Step 3 — rank & pick top N

Score remaining jobs (0-100) by skill-overlap with `cv.txt`. Use Gemini default for routine cases. **Escalate to `mid`** (Sonnet 4.6) only when ≥3 candidates have ambiguous fit signals (similar scores within 10pts, conflicting JD signals). Sort desc, take top `limit` (default 5).

### Step 4 — gather company context (per job)

For each of the top N:
1. Resolve `company_domain` from listing URL or `jobmcp__list_companies`.
2. Call these in parallel and assemble locally:
   - `company__company_profile(domain=<domain>)` — structured profile (description, industry, products, hq, size, careers_url, linkedin_url) with `confidence` and `sources`. Backed by an OpenRouter extractor (`OPENROUTER_EXTRACTION_MODEL`) and synthesizer (`OPENROUTER_QUALITY_MODEL`); results cached server-side.
   - `company__recent_news(company=<name>, days=30)` — last-30d news from Tavily.
   - `company__linkedin_company_lookup(company=<name>)` — official LinkedIn page candidates ranked from search snippets.
3. **Cache hint:** if you've already pulled this company today, use `company__cached_company_results(company=<name>)` first; only call the live tools when cache misses or `confidence < 0.6`.
4. **Degradation rules:** if `company_profile` returns `confidence < 0.3` or errors, log the warning to chat and proceed with whatever subset of tools succeeded. Don't block the pipeline on a single provider.
5. Pass the assembled `{profile, recent_news, linkedin}` object as the company context for step 5. Don't store secrets.

### Step 5 — build briefing (heavy)

For each job, call the model with alias **`heavy`** **exactly once** via:
```
openclaw infer model run --model heavy \
  --prompt "<assembled prompt>" --json
```
**Do not hardcode a concrete model id here.** The `heavy` / `mid` / `cheap` aliases are resolved at runtime against `~/.openclaw/openclaw.json`, which is rewritten by `~/jobsearcher-deploy/switch-openrouter-tier.sh free|paid`. On the `paid` tier `heavy` resolves to Opus-class; on `free` it resolves to a free OpenRouter model (currently Qwen 80B). Hardcoding a model id bypasses the tier toggle and will 402 on free-tier accounts.

The prompt contains: cv.txt, the listing, the company context object (the `overview` from step 4), the target profile, and the briefing template (sections from `MEMORY.md → Per-job briefing pipeline → step 4`).

Expect ~600-word markdown back. Do not retry `heavy` for revisions — if output is malformed, retry once on `mid` with the same prompt; if still bad, surface the raw context to chat and abort that job's briefing.

### Step 6 — deliver

For each completed briefing (skipped if `dry-run=true`):
1. **Email:** `gmail__send_email(subject="[JobBrief] <Role> @ <Company> — <match%>", body_markdown=<briefing>)`. Capture `provider_message_id`.
2. **Slack heads-up:** `slack__conversations_add_message(channel_id="C0B0K3X8VND", text="<Role> @ <Company> — <match%> match. Briefing in inbox. Reply yes/no/skip.")`. Capture the Slack `ts`.
3. Append `{job_id, role, company, match_pct, gmail_id, slack_ts, sent_at, via_fallback?}` to today's `memory/YYYY-MM-DD.md` under a `briefings:` section. Include `via_fallback: mock` only when this job came from the Step 1c fallback path; otherwise omit the field.
4. **Mark processed:** `exec("node /home/node/.openclaw/workspace/scripts/processed.js add <canonical_id>")`. Use the canonical id from Step 1 (mock backend: jobmcp id; real backend: `{source}:{source_job_id}` or url-hash fallback). Do this only after the email + Slack post both succeed — partial sends should NOT mark the job processed.

### Step 7 — wait for reply (only if not dry-run)

This skill **returns** after step 6. The "yes/no/skip" reply path is handled in a separate skill (`job-search-reply`, see below) so the user can reply hours later without keeping this pass open.

## Reply handler (separate flow)

When the user types **"check job-search replies"** (or runs this skill again with `mode=replies`):
1. For each entry in today's `memory/YYYY-MM-DD.md → briefings` without a `decision` field, fetch the Slack thread starting at `slack_ts` via `slack__conversations_replies`.
2. Parse the first reply from Nikita (case-insensitive `yes|no|skip`).
3. On **yes**:
   - **mock backend** → call `jobmcp__submit_mock_application(job_id)`, post a confirmation reply in the Slack thread, and write `decision=apply` + `application_id` to the briefing entry.
   - **real backend** → no mock-apply tool exists. Post a Slack reply with the listing's `application_url` ("Apply manually here: <url>") and write `decision=apply_manual` + `application_url` to the briefing entry.
4. On **no** / **skip**: write `decision=<value>`, post a brief ack in thread.

## Output to chat

Throughout the pass, narrate progress as bullet lines (the user prefers verbose step-by-step output):
- `[step 1] searched jobmcp (mock): 17 raw → 9 after dedupe` (or `searched realjobmcp (real): 12 raw → 8 after dedupe` for `backend=real`)
- `[step 2] hard-filter: 9 → 5 (rejected 3 for geo, 1 for dealbreaker)`
- `[step 4] gathering company context: Acme Corp (acme.io)`
- `[step 5] heavy synthesis: ✓ briefing 612 words`
- `[step 6] email sent (id b1e...) + Slack heads-up posted (ts 1761...)`

End with a one-line summary: `done. <N> briefings sent, awaiting Slack replies.`

## Failure modes & guardrails

- **OpenRouter 402 (insufficient credit):** the active tier has no balance. Report once to chat, retry the step on Gemini default (no model flag), and remind the user they can flip tiers via `bash ~/jobsearcher-deploy/switch-openrouter-tier.sh free|paid`. Never silently swallow.
- **Gemini 429:** openclaw auto-falls-back to `cheap` (configured). If `cheap` also 402s, abort the pass cleanly, report to chat, leave today's `memory/YYYY-MM-DD.md` consistent (don't half-write briefings).
- **`company_profile` timeout / low confidence (<0.3):** log a warning to chat and proceed with `recent_news` + `linkedin_company_lookup` only. Don't block the pipeline on a single provider.
- **gmail-mcp returns sandbox-restriction error:** that's the Resend `onboarding@resend.dev` sender only delivering to the account owner. Already expected for chucheloff@gmail.com. If the recipient differs, surface the error.
- **Slack API rate-limit:** back off and retry once after 5s. Don't loop.
- **Same job already briefed (any prior day):** skip silently (its id is in the Valkey `jobsearch:processed` set; loaded into `processed_ids` in Step 0).
