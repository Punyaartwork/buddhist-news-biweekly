# buddhist-news-biweekly

Biweekly scan of global Buddhist news for the **เด็กพุทธ** content team. A skill in this repo collects verified items every 2 days, commits a structured Brief into `briefs/`, and a GitHub Actions workflow dispatches the Brief to a shared Telegram channel.

Same Telegram channel as `Punyaartwork/dailyainews` — messages are prefixed with 🪷 to distinguish from AI Briefs (🤖).

## How it works

```

cron (every 2 days, 06:00 Asia/Bangkok)
    │
    ▼
.github/workflows/scheduler.yml  ──▶ invokes the skill runner
    │
    ▼
.claude/skills/buddhist-news-biweekly/SKILL.md   (7 steps)
    │
    ├─ reads reference/defaults.json
    ├─ reads reference/trusted-sources.md
    ├─ reads reference/category-schema.md
    └─ reads reference/skill-routing-map.md
    │
    ▼
commit Brief to briefs/YYYY/MM/YYYY-MM-DD.md   (via GitHub MCP)
    │
    ▼
.github/workflows/telegram-notify.yml fires on push to briefs/**
    │
    ▼
🪷 Telegram channel notification (shared with dailyainews)

```

## Layout

```

buddhist-news-biweekly/
├── .claude/skills/buddhist-news-biweekly/
│   ├── SKILL.md
│   └── reference/
│       ├── defaults.json
│       ├── trusted-sources.md
│       ├── category-schema.md
│       └── skill-routing-map.md
├── briefs/
│   └── YYYY/MM/YYYY-MM-DD.md
├── .github/workflows/
│   ├── scheduler.yml
│   └── telegram-notify.yml
└── README.md

```

## Secrets required (GitHub Actions)

| Secret | Purpose |
|---|---|
| `TELEGRAM_BOT_TOKEN` | Same bot token as `dailyainews`. |
| `TELEGRAM_CHAT_ID` | Same chat ID as `dailyainews`. |

Copy values from `Punyaartwork/dailyainews` repo settings.

## Hard rules

- No Bash / shell / git CLI inside the skill — commit via GitHub MCP only.
- Verification tiers: WebFetch + 48h-fresh preferred, WebSearch snippet from a trusted domain as fallback.
- Idempotency: identical Brief content = NO-OP, no re-commit.
- Controversies category defaults OFF.
- Telegram dispatch happens via the workflow, never inside the skill.

## First-run checklist

1. ✅ Repo created.
2. ✅ Files placed at correct paths.
3. ⬜ Set repo secrets `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID`.
4. ⬜ Wire up the actual skill runner inside `.github/workflows/scheduler.yml` (mirror dailyainews approach).
5. ⬜ Test by running scheduler manually via `workflow_dispatch`.
