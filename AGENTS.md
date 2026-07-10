# AGENTS.md

This file provides guidance to AI coding agents (Hermes, Claude Code, etc.) working in this repository. It is the single source of truth for project context, patterns, and conventions.

## Project

Iris — an AI email assistant powered by Hermes Agent. Iris screens your inbox, categorizes emails, drafts replies in your voice, and delivers twice-daily briefs summarizing everything non-urgent. Think of it as an open-source, self-hosted Cora alternative built on Fastmail + Hermes.

- **Runtime**: Hermes Agent (Python), Fastmail MCP tools
- **Language**: Markdown (skills/plans), shell scripts (cron jobs), Python (where needed for processing)
- **Email backend**: Fastmail (via MCP tools: search, read, draft, archive, etc.)
- **Delivery**: Discord and Telegram (via Hermes platform adapters)
- **Scheduling**: Hermes cron jobs
- **Deployment**: Runs on existing Hermes Agent VPS (no new infra needed)
- **Single developer**: NamalD on GitHub

## Architecture

Iris is not a traditional web app — it's a set of Hermes skills and cron jobs that use the existing Fastmail MCP integration. No new servers, no new databases, no new deployments.

### Component Map

```
┌──────────────────────────────────────────────────┐
│                    Hermes Agent                    │
│                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │  Skills  │  │Cron Jobs │  │ Fastmail MCP     │ │
│  │          │  │          │  │ (search/read/    │ │
│  │ iris-   │  │ morning  │  │  draft/archive/   │ │
│  │ inbox   │  │ brief    │  │  update/delete)   │ │
│  │ iris-   │  │ afternoon│  │                   │ │
│  │ draft   │  │ brief    │  │                   │ │
│  │ iris-   │  │ inbox    │  │                   │ │
│  │ brief   │  │ triage   │  │                   │ │
│  └──────────┘  └──────────┘  └──────────────────┘ │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │         Delivery (Discord / Telegram)         │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

### Skills (Hermes skills loaded on demand)

| Skill | Purpose |
|-------|---------|
| `iris-inbox` | Triage inbox: categorize emails, decide what needs reply vs brief |
| `iris-draft` | Draft replies to emails in the user's voice |
| `iris-brief` | Generate the twice-daily brief summary |
| `iris-configure` | Configure Iris preferences (voice, categories, rules) |

### Cron Jobs (scheduled via Hermes cronjob tool)

| Job | Schedule | Purpose |
|-----|----------|---------|
| Morning Brief | 8:00 AM daily | Summarize overnight emails, deliver brief |
| Afternoon Brief | 4:00 PM daily | Summarize day's emails, deliver brief |
| Inbox Triage | Every 15 min (optional) | Screen new emails, archive non-urgent, flag urgent |

### Data Flow

1. **Inbox Triage** (cron or on-demand): `search_email` → categorize with LLM → urgent stay in inbox, others archived to "Iris Briefed" folder
2. **Draft Replies** (on-demand): user asks "draft replies" → `read_email` on urgent emails → LLM drafts in user's voice → `draft_email` to Drafts folder
3. **Brief Generation** (cron): `search_email` in "Iris Briefed" folder since last brief → LLM summarizes → deliver to Discord/Telegram
4. **Configuration** (conversational): user tells Iris preferences → saved to memory/skill → used in future triage/drafts

### Key Design Decisions

- **Never sends email autonomously** — drafts go to Drafts folder only, user reviews before sending
- **Folder-based tracking** — Iris creates an "Iris Briefed" folder; archived non-urgent emails go there. This keeps the actual inbox clean and gives the user a way to review what Iris did
- **LLM-driven categorization** — no ML training needed. Each email is categorized by the LLM at triage time based on user-defined rules
- **Memory-backed preferences** — user preferences (voice style, importance rules, category definitions) stored in Hermes memory, not a separate config file

## Commands

```bash
# No build step — Iris is a collection of Hermes skills and cron jobs
# Skills are loaded by Hermes at runtime

# Test email connectivity
hermes run "search my inbox for recent emails"

# Manually trigger a brief
hermes run "generate my email brief using iris-brief"

# Manually triage inbox
hermes run "triage my inbox using iris-inbox"
```

## Commit conventions

Follow the same conventions as Warframe TODO Tracker:

```
[Tag] Short description (#issue-number)

- Bullet points of what was done

Closes #N
```

Tags: `[Plan]`, `[Work]`, `[Review]`, `[Compound]`, `[Feature]`, `[Fix]`, `[Skill]`, `[Config]`

## Project board discipline

Same as Warframe TODO Tracker:
- Issues move Todo → In Progress when picked up
- `Closes #N` in commits auto-moves to Done
- Board must match reality after every commit

## Known gotchas

- Fastmail MCP tools use either "folders" or "labels" mode — Iris assumes folders mode
- **Fastmail MCP cannot create folders** — use the Archive folder for triage by default. If the user wants a separate "Iris Briefed" folder, they must create it manually in the Fastmail web UI, then update iris-inbox to use `update_email(folder="Iris Briefed")` instead of `archive_email`.
- Hermes cron jobs in recurring mode need careful idempotency — don't brief the same emails twice. Track `Iris: last brief` and `Iris: last triage` timestamps in memory.
- Discord doesn't render markdown tables — briefs must use emoji section headers (📰 📦 🔔) and `•` bullet lists. Max ~1800 chars per message (Discord limit is 2000).
- The `search_email` tool has a `receivedAt` field — use `after`/`before` with relative dates for time-window queries. Always use time windows, never search the full inbox (can have thousands of emails).
- Draft emails via MCP go to the Drafts folder; the user sends them manually through Fastmail UI. Never use any send capability.
- **Hermes memory budget**: ~2,200 chars. Iris entries must stay under ~90 chars each. Use pipes as delimiters, no spaces between fields. Consolidate entries when approaching limit — one `memory(operations=[...])` batch call is safer than multiple `add` calls.
- **Voice detection > self-reporting**: Auto-detect writing voice from sent mail (search "from:me in:sent", read 3-5 bodies) rather than asking the user to describe their style. Self-reported style is unreliable — real emails are the ground truth.
- **Skills are tested by running against real data**, not unit tests. After creating a skill, do a dry run against actual Fastmail data to verify correctness before committing.
- **GitHub project board API** requires explicit field/option IDs for status changes — the `gh project item-edit` syntax differs between user projects and org projects. Prefer closing issues via `Closes #N` in commits for automatic board movement.
- **GH CLI label creation**: Labels must exist before using them in `gh issue create --label`. If a label is missing, create it first or use an existing one.
