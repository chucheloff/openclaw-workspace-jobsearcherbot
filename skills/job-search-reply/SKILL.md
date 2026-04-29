---
name: job-search-reply
description: Reply-handler companion to the `job-search` skill. Polls today's briefings (recorded in `memory/YYYY-MM-DD.md`), pulls each Slack thread for Nikita's reply (yes/no/skip), and on a "yes" submits a mock application via jobmcp + posts confirmation. Trigger phrase: "check job-search replies" (with optional `date=YYYY-MM-DD` to target a past day).
metadata:
  { "openclaw": { "emoji": "📬", "requires": { "config": [] } } }
---

# job-search-reply

Companion to `job-search`. Drains pending decisions from today's (or a target date's) briefings file, applies the corresponding action, and writes the decision back so the same briefing isn't processed twice.

## When to invoke

- User says **"check job-search replies"** (with or without modifiers).
- Optional modifiers:
  - `date=YYYY-MM-DD` — process a different day's briefings file (default: today).
- The companion `job-search` skill *returns* after step 6 of its pipeline; it does **not** wait for replies. This skill is how those waiting briefings get resolved later.

Do **not** auto-trigger this skill. It's user-initiated.

## Required context (read once)

1. `memory/<date>.md` — the briefings file written by `job-search`. Each briefing entry contains `{job_id, role, company, gmail_id, slack_ts}` and (after this skill runs) a `decision=` field.
2. Nothing else. CV/MEMORY.md/skill profile are not needed here — this skill only routes pre-existing decisions.

## Connected MCP servers (already wired)

| Server  | Tools                                                          |
| ------- | -------------------------------------------------------------- |
| `slack` | `conversations_replies`, `conversations_add_message` (channel `C0B0K3X8VND` = `#job-search`) |
| `jobmcp`| `submit_mock_application`                                      |

Tools are namespaced by server name in openclaw — e.g. `slack__conversations_replies`, `jobmcp__submit_mock_application`.

## Pipeline

### Step 0 — bootstrap

Resolve target date (default today). Read `memory/<date>.md`. Parse all briefing entries. Filter to those **without** a `decision` field. If none, print `no pending replies for <date>` and exit cleanly.

### Step 1 — fetch Slack thread per pending briefing

For each pending briefing:
1. Call `slack__conversations_replies(channel="C0B0K3X8VND", ts=<slack_ts>, limit=20)`.
2. Find the **first** reply from Nikita (skip the bot's original post; match by user id if known, otherwise the first non-bot message). Lowercase it.
3. Match against `yes` / `no` / `skip` (case-insensitive, allow leading whitespace and trailing punctuation). Anything else → leave as **pending** (do NOT default to skip; user might still be deciding).

Cap concurrent Slack calls at 3 to be polite to the API.

### Step 2 — apply each decision

- **`yes`**:
  1. Call `jobmcp__submit_mock_application(job_id=<id>)`. Capture `application_id` from the response.
  2. Post `slack__conversations_add_message(channel="C0B0K3X8VND", thread_ts=<slack_ts>, text="✅ Submitted application (id: <application_id>). Good luck!")`.
  3. Append `decision=apply` and `application_id=<id>` to the briefing entry in `memory/<date>.md`.

- **`no`**:
  1. Post `slack__conversations_add_message(channel="C0B0K3X8VND", thread_ts=<slack_ts>, text="👍 Skipping this one. Noted.")`.
  2. Append `decision=no` to the briefing entry.

- **`skip`**:
  1. Post `slack__conversations_add_message(channel="C0B0K3X8VND", thread_ts=<slack_ts>, text="⏭️ Skipped — won't surface again today.")`.
  2. Append `decision=skip` to the briefing entry.

- **No reply yet** (no parseable yes/no/skip in the thread): leave the entry untouched. It'll be picked up on the next invocation.

### Step 3 — summary

End with a one-line summary:
`done. <N_apply> applied, <N_no> declined, <N_skip> skipped, <N_pending> still waiting.`

## Output to chat

Per-briefing narration (verbose):
- `[reply] job-016 (JPMorgan): yes → submit_mock_application id=APP-...; thread ack posted`
- `[reply] job-022 (ING): skip → thread ack posted`
- `[reply] job-031 (Acme): no reply yet, leaving pending`

## Failure modes & guardrails

- **`submit_mock_application` returns error:** post a thread reply mentioning the failure, leave `decision=` blank so the user can retry. Don't write `decision=apply` on a failed submission.
- **Slack API rate-limit (HTTP 429):** back off 5s, retry once, then abort cleanly with a chat note. Process whatever briefings did complete.
- **Multi-message ambiguity in thread:** trust the *first* user reply only. If the user changes their mind, they should say so explicitly in a new thread; don't try to parse "actually, no, wait, yes."
- **Missing briefings file for `<date>`:** print `no briefings file for <date>` and exit zero (this isn't an error, just nothing to do).
- **Same briefing already has `decision=`:** skip silently. Idempotent re-runs are explicitly supported.
