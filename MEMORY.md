# MEMORY.md — Long-term curated memory

## About Nikita (the user)

Software Developer / Data Engineer, 5+ years in fintech. Almaty, Kazakhstan. See `USER.md` for contact info, `cv.txt` for full CV.

### Skill profile (from CV)

- **Primary languages:** Python (FastAPI, DLT, pypdf-class data tooling), C# (AvaloniaUI desktop apps), VBA (Excel add-ins), SQL.
- **Data engineering:** ETL with DLT, ClickHouse (OLAP), MinIO (object storage), multi-source feed consolidation (stock exchange data).
- **AI / LLM:** Self-hosted LLM serving via FastAPI + Ollama; production LLM inference pipelines.
- **Infra:** Docker, Git, ArgoCD.
- **QA background:** Mobile test automation with Appium + WebDriverIO (JS), iOS + Android.

### Experience timeline

- **2020-present:** RUDATA (Interfax Digital Solutions), Moscow — Software Developer / Data Engineer. Cross-platform financial reporting tool (AvaloniaUI/C#), ETL pipelines (Python/DLT consolidating exchange feeds), Excel VBA market-analysis add-in.
- **2023-2025:** AV3.Studio, Moscow — Freelance Backend Developer. FastAPI services serving self-hosted LLMs for video production teams; Ollama integration.
- **2020-2021:** BetPlanet, Cyprus — QA Automation Engineer. Mobile test automation suite (Appium/WebDriverIO).

### Education

- MISiS (National University of Science and Technology), Moscow, 2019-2022. Engineering Cybernetics — Applied Mathematics & Advanced Science-Based Software Algorithms.

### Job-search target profile (CONFIRMED by Nikita 2026-04-28)

**Target roles (ranked, most-wanted first):**
1. Senior Data Engineer
2. Backend Engineer (Python)
3. Platform / Infra Engineer

**Roles to avoid:**
- Frontend-only
- Pure ML Research
- Java / .NET-heavy backend

**Geo / work mode (HARD filters):**
- `remote_ok: true` — remote acceptable
- `remote_only: false` — on-site / hybrid OK if in allowed location
- Allowed countries: Kazakhstan, Germany, Netherlands, UK
- Allowed cities (narrower preference): Almaty, Berlin, Amsterdam, London
- `visa_sponsorship_required: true` — reject roles that won't sponsor (except Kazakhstan)
- `timezone_overlap_hours: 4` — min 4h overlap with Asia/Almaty (UTC+5)

**Seniority:** mid → staff (reject junior, principal+).

**Compensation (gross USD/yr equivalent):**
- min_base_usd: 60000
- preferred_base_usd: 90000
- equity_ok: true
- contract_ok: false (no B2B / contractor)

**Language:**
- Required: English
- Nice-to-have: Russian, German

**Soft preferences (use as scoring signals, not filters):**
- Fintech / capital-markets domain a plus
- Python + ClickHouse / DLT / FastAPI stack a strong match
- Avoid 24/7 on-call rotations
- Avoid crypto-trading shops

**DEAL-BREAKERS (any one match → auto-reject):**
- Unpaid / equity-only
- Requires relocation to US (visa pain)
- Gambling / adult industry


## Operating context

- Slack channel for job posts: `#job-search` / channel_id `C0B0K3X8VND`.
- Job-source MCP: `jobmcp` (tools: search_jobs, get_job, list_companies, submit_mock_application, etc).
- Slack MCP: read tools + `conversations_add_message` for posting.
- Gmail notifications: not yet wired (planned).


## Job-search pipeline (the briefing flow)

For each promising job listing surfaced by `jobmcp__search_jobs`, do **NOT** post the raw listing. Instead, build a **briefing** and send it to Nikita via Gmail (Slack is for the short heads-up + yes/no decision).

### Per-job briefing pipeline

1. **Fetch the job:** `jobmcp__search_jobs` → pick top matches via match score (see Skill profile in this MEMORY.md).
2. **Resolve company domain** from the job listing (employer name → domain). If the listing has a company URL, use it; otherwise infer.
3. **Gather company info** via `companymcp` (registered as `company` in openclaw):
   - `company__company_profile(domain=...)` → mission, products, size, what they do.
   - `company__recent_news(company=..., days=30)` → last 30 days of news (funding, layoffs, product launches).
   - `company__linkedin_company_lookup(company=...)` → official LinkedIn page, headcount, industry.
   - (Optional) `company__linkedin_lookup(name=<hiring manager if known>)`.
4. **Build the briefing** (markdown, max ~600 words). Sections:
   - **Role:** title, company, location, comp if listed, source URL.
   - **Match score & why:** % match + 2-line rationale.
   - **Skills matched** — bulleted intersection of Nikita's skills (cv.txt / MEMORY.md) and job requirements. Quote the JD line for each match.
   - **Skill gaps** — what the JD asks for that Nikita doesn't have, AND what to learn/sharpen before the interview (concrete: "brush up on X by reading Y").
   - **Company snapshot** — 3-5 bullets from `company_profile` + recent news (1-2 newsworthy items max, with source link + date).
   - **CV adjustments** — 2-4 specific edits to tailor cv.txt for *this* role (re-order bullet points, swap keywords, surface relevant projects). Don't rewrite the whole CV; just the deltas.
   - **Recommendation** — APPLY / SKIP / NEEDS-NIKITA-INPUT, one-line reason.
5. **Send the briefing via Gmail** to chucheloff@gmail.com using `gmail__send_email(subject, body_markdown)` (Resend backend; sender shows as `JobSearcherBot <onboarding@resend.dev>`). Subject: `[JobBrief] <Role> @ <Company> — <match%>`.
6. **Post a short heads-up to Slack** (`#job-search`, channel C0B0K3X8VND): one line — role, company, match%, "briefing in your inbox, reply yes/no/skip".
7. **On Nikita's "yes" reply** in Slack: call `jobmcp__submit_mock_application` (real apply path TBD), then post confirmation to Slack.

### Caching / idempotency

- Don't re-brief the same job twice in a session — keep a list of already-processed job IDs in `memory/YYYY-MM-DD.md`.
- `companymcp` caches via Valkey (db 1) so repeated calls are cheap.


## Model tiering policy (token-cost optimization)

Default agent model is `google/gemini-2.5-flash-lite` (free-tier Gemini API key) — handles all routine orchestration: reading MEMORY.md, choosing which jobs to investigate, driving MCP calls, parsing tool outputs, deciding which Slack channel to post to, writing the short Slack heads-up.

For high-leverage steps that warrant smarter (paid) models, escalate explicitly via `openclaw infer model run --model <id> --prompt ...`. The aliases are configured but `infer` does **not** expand them — pass full provider/model IDs:

| Tier   | Alias  | Model id                                   | Use for                                                                 |
| ------ | ------ | ------------------------------------------ | ----------------------------------------------------------------------- |
| cheap  | cheap  | `openrouter/anthropic/claude-haiku-4.5`    | Bulk classification, JD <-> skill matching, geo/dealbreaker filtering when Gemini is rate-limited or output quality matters but cost matters more. |
| mid    | mid    | `openrouter/anthropic/claude-sonnet-4.6`   | Job-by-job match-score reasoning, skill-gap analysis, CV-adjustment suggestions. |
| heavy  | heavy  | `openrouter/anthropic/claude-opus-4.7`     | **Final briefing synthesis only** (the markdown body that gets emailed). Never use for upstream gathering. |

### Decision rules

- **Default path:** Gemini drives everything end-to-end. Only escalate when the step's output goes directly to the user (briefing) or when match-score correctness materially changes which jobs get briefed.
- **Briefing synthesis:** always use `heavy` (Opus 4.7). Pipe the gathered context as the prompt; expect ~600-word markdown back. One Opus call per job, max.
- **Match-score / skill-gap reasoning:** use `mid` (Sonnet 4.6) when >=3 candidate jobs share ambiguous fit signals; otherwise let Gemini decide.
- **Bulk filtering of >20 jobs:** use `cheap` (Haiku 4.5) batched 10 at a time.
- **Fallback chain:** Gemini default -> on 429/quota error openclaw auto-falls-back to `cheap`. Manual escalation overrides this.

### Cost guardrails

- **OpenRouter requires paid credit** (https://openrouter.ai/settings/credits). Until topped up, all `cheap`/`mid`/`heavy` calls return HTTP 402. Gemini-only path still works but is rate-limited.
- Don't loop on `heavy` — one Opus call per briefing. If the briefing needs revision, do the edit on `mid` or Gemini.
- Don't call `infer` from inside another `infer` (no recursion).


## On-demand job-search trigger

The full pipeline (steps 1-7 above) is encoded as a workspace skill at `~/.openclaw/workspace/skills/job-search/SKILL.md` (registered name: `job-search`).

Invoke when Nikita says **"run job-search pass"** (optionally with `query=`, `limit=N`, or `dry-run=true` modifiers). Reply handler runs on **"check job-search replies"**.

Do NOT auto-trigger this skill on heartbeat or session start. It's user-initiated only.
