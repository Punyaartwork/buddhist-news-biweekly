# Skill: buddhist-news-biweekly

Scan global Buddhist news every 2 days, verify against trusted sources, and commit a structured Brief to this repo. A GitHub Actions workflow dispatches the Brief to the shared Telegram channel afterwards.

This skill mirrors the structure of `daily-ai-news` but is tuned for the Buddhist content team (เด็กพุทธ) at biweekly cadence with a 48-hour scan window.

---

## Hard Constraints (do not override)

- **No Bash, no shell, no git CLI.** All file writes to the repo go through the GitHub MCP connector.
- **Verification tiers**:
  - Tier 1 — `WebFetch` returns 2xx AND article timestamp is within `window_hours` (default 48h). Preferred.
  - Tier 2 — `WebSearch` snippet from a domain listed in `reference/trusted-sources.md`. Allowed only when WebFetch is blocked. Never cite a URL that is not in the search results for a trusted domain.
- **If the whole runtime is WEBFETCH_BLOCKED**, commit anyway and tag the commit message with `[verify=search]`.
- **Timezone**: Asia/Bangkok.
- **GitHub connector missing → abort** (do not fall back to Bash, do not write locally).
- **Do NOT attempt Telegram push.** Telegram dispatch is handled by `.github/workflows/telegram-notify.yml` after the commit.
- **Idempotency (Step 5a)**: if the candidate Brief is byte-identical to the file already at the target path, do NOT re-commit. Output NO-OP in the Step 7 report.
- **`include_controversies` defaults to `false`.** Skip any candidate tagged Controversies unless the user explicitly overrides per-run.

---

## Inputs

- **Required**: GitHub MCP connector authenticated.
- **Read**: `reference/defaults.json`, `reference/trusted-sources.md`, `reference/category-schema.md`, `reference/skill-routing-map.md`.
- **No env paste needed**: identity (owner, repo, branch) comes from `reference/defaults.json`.

---

## 7 Steps (execute in order)

### Step 1 — Load configuration

Read all four reference files via the GitHub MCP connector (or Read tool when running from a local clone). Resolve:

- `owner`, `repo`, `branch` → for all commits.
- `window_hours` → freshness window for Tier 1 verification.
- `schedule`, `timezone` → for sanity check (the cron is enforced by `.github/workflows/scheduler.yml`, not by this skill).
- `min_relevance_score`, `max_primary_picks`, `fallback_no_news`, `include_controversies`.
- `output_path` template → resolve `{year}`, `{month}`, `{date}` against today in Asia/Bangkok.
- `telegram_prefix` → only used inside the Brief header so the workflow can route formatting.

If any required field is missing → abort with reason.

### Step 2 — Compute the scan window

- `now_bkk` = current time, Asia/Bangkok.
- `window_start` = `now_bkk - window_hours` (default 48h).
- `window_end` = `now_bkk`.
- Record both timestamps — they go into the Brief header for transparency.

### Step 3 — Scan trusted sources

For every domain listed in `reference/trusted-sources.md`:

1. WebFetch the source's news index / RSS / category landing page.
2. Extract candidate items with `{title, url, published_at, source_domain, snippet}`.
3. Drop items where `published_at` is outside `[window_start, window_end]`.
4. Tag each remaining item with one of the 7 categories from `reference/category-schema.md`. If a candidate fits Controversies and `include_controversies` is `false`, drop it now.

Collect everything into a single `candidates[]` list.

If a source's WebFetch returns non-2xx or is blocked:
- Try Tier 2: WebSearch for `site:<domain> <category-keywords> after:<window_start_date>`.
- Only keep results whose URL is on that same trusted domain.

### Step 4 — Verify each candidate

For every candidate, attempt Tier 1 first:

1. WebFetch the article URL. If 2xx AND timestamp parses inside the window → mark `verify=fetch`.
2. Else fall back to Tier 2: confirm the article appears in a WebSearch result list and the URL's domain is in `trusted-sources.md` → mark `verify=search`.
3. Else drop the candidate and log it under `SKIPPED` with reason.

If the runtime-level WebFetch is fully blocked, mark all surviving candidates `verify=search` and remember to add `[verify=search]` to the commit message in Step 5.

### Step 5 — Filter, rank, and format the Brief

1. Score each verified candidate on relevance (1–10) using these factors:
   - Cross-cultural reach (does it interest Thai audiences who don't follow global Buddhism?)
   - Novelty (would the typical เด็กพุทธ viewer say "ไม่เคยรู้เลย"?)
   - Fit with one of the registry skills (see `skill-routing-map.md`).
   - Brand tone fit (warm, non-preachy, story-driven).
2. Drop anything below `min_relevance_score`.
3. Sort by score descending.
4. Promote the top N (≤ `max_primary_picks`) to **PRIMARY PICKS**; the rest become **RUNNERS-UP**.
5. Render the Brief using the template in `reference/category-schema.md` → "Brief Template" section.

If after filtering there are zero primary picks:
- If `fallback_no_news` is `true` → produce a minimal Brief with header `NO-QUALIFYING-NEWS: true` and an empty primary section, and continue to Step 5a.
- If `false` → abort with reason "no qualifying news".

### Step 5a — Idempotency check before commit

1. Read the current file at `output_path` from `owner/repo@branch` via GitHub MCP.
2. If it exists and its content equals the candidate Brief byte-for-byte → **NO-OP**. Skip Step 6 and jump to Step 7.
3. If it exists and differs → keep the returned `sha` and pass it to the GitHub MCP update call in Step 6.
4. If it does not exist → create new file in Step 6.

### Step 6 — Commit via GitHub MCP

- Commit message format:
  `buddhist-brief: {YYYY-MM-DD} ({primary_count} picks){verify_tag}`
  where `verify_tag` is `" [verify=search]"` if Tier 2 was forced runtime-wide, otherwise empty.
- File path: resolved `output_path`.
- Use the GitHub MCP file-create or file-update tool (with `sha` from Step 5a if updating).
- Capture the returned commit `sha` and `permalink`.

### Step 7 — Report

Emit a single fenced block in this exact shape:

```
Buddhist News Brief — {YYYY-MM-DD}
Status: {COMMITTED | NO-OP | NO-QUALIFYING-NEWS}
Path: {output_path}
Short SHA: {first-7-of-sha}
Permalink: {permalink}
Verify tier: {fetch | search | mixed}
Primary picks: {n}
Runners-up: {n}
Skipped: {n}
Telegram: dispatched by workflow
```

If NO-OP: keep the same fields but show the existing file's short SHA and permalink; primary/runner counts come from the existing file.

---

## Errors and aborts

| Condition | Action |
|---|---|
| GitHub MCP missing or unauthenticated | Abort. Tell the user to `/mcp` authenticate `plugin:engineering:github`. |
| `reference/defaults.json` missing or invalid JSON | Abort with file path. |
| All sources unreachable AND WebSearch returns nothing for trusted domains | Abort with "no verifiable candidates". |
| `fallback_no_news: false` and zero passes filter | Abort with "no qualifying news today". |
| Commit attempt returns 4xx from GitHub MCP | Surface raw error, do not retry silently. |
