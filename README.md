# Iris 🌐

AI email assistant powered by [Hermes Agent](https://hermes-agent.nousresearch.com/). Iris screens your inbox, drafts replies in your voice, and delivers twice-daily briefs summarizing everything non-urgent.

Think of it as an open-source, self-hosted [Cora](https://cora.computer/) alternative — but for Fastmail, delivered through Discord and Telegram.

## What Iris Does

- **Inbox Triage** — categorizes every incoming email as urgent or non-urgent. Urgent stays in your inbox. Everything else gets archived to an "Iris Briefed" folder.
- **Draft Replies** — writes reply drafts in your voice for emails that need a response. Drafts go to your Drafts folder — you review and send.
- **Twice-Daily Brief** — morning and afternoon summaries of everything non-urgent: newsletters, notifications, receipts, calendar invites. Delivered to Discord and Telegram.
- **Calendar Summary** — includes upcoming meetings, RSVPs, and invites in your brief.
- **Conversational** — chat with Iris to adjust preferences, ask about emails, or trigger actions. "What did Amazon ship?" "Draft a reply to Sarah."

## How It Works

Iris is not a standalone app — it's a set of [Hermes skills](skills/) and [cron jobs](cron/) that use the existing [Fastmail MCP integration](https://hermes-agent.nousresearch.com/docs). No new servers, no new databases.

```
Your Inbox → Iris (Hermes) → Urgent stays in inbox
                            → Drafts replies for you
                            → Archives rest to "Iris Briefed"
                            → Brief delivered to Discord/Telegram
```

## Project Status

🚧 **Planning Phase** — [see the plan](docs/plans/)

## Structure

| Directory | Purpose |
|---|---|
| `skills/` | Hermes skills (iris-inbox, iris-draft, iris-brief, iris-configure) |
| `cron/` | Cron job definitions (morning brief, afternoon brief, inbox triage) |
| `docs/plans/` | CE implementation plans |
| `docs/designs/` | Architecture decisions, feature analysis |
| `AGENTS.md` | Single source of truth for AI coding agents |

## Setup

```bash
# Coming soon — see docs/plans/ for implementation roadmap
```

## License

MIT
