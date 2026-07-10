---
name: iris-brief
description: "Generate a twice-daily email brief summarizing non-urgent emails. Groups by category, delivers a scannable Discord-friendly summary. Designed for cron scheduling (8am morning, 4pm afternoon)."
trigger: |
  User asks for "email brief", "what did I miss", "summarize my emails",
  or this skill is loaded by the scheduled morning/afternoon cron jobs.
---

# Iris Brief — Twice-Daily Email Summary

Generate a scannable summary of all non-urgent emails received since the last brief. Designed to be read in 30 seconds on Discord.

## When to Use

- **Cron-triggered**: 8am (morning brief) and 4pm (afternoon brief) daily
- **On-demand**: User says "generate my brief", "what's in my email", "summarize"
- After inbox triage to see what was archived

## Workflow

### Step 1: Load Preferences

Read from Hermes memory:
- `Iris categories` — how to group emails
- `Iris delivery` — timezone for time window calculations
- `Iris: last brief — [timestamp]` — when the last brief was generated

### Step 2: Determine Time Window

If `Iris: last brief` exists: search for emails received between then and now.
If first run (no prior brief): search for emails from the last 12 hours.

```bash
# Example queries:
search_email("in:archive after:2026-07-10T08:00:00+01:00 before:2026-07-10T16:00:00+01:00", limit=50)
```

Note: Emails are in the Archive folder (moved there by inbox triage). If the user has an "Iris Briefed" folder, use that instead.

Also include urgent emails from inbox that were read/processed (for completeness):
```bash
search_email("in:inbox is:read after:[last_brief]", limit=20)
```

### Step 3: Group by Category

Categorize each email based on sender, subject, and content. Use the categories from Iris preferences:

| Category | Matches |
|----------|---------|
| **Newsletters** | Bulk senders, unsubscribe links, "[newsletter]", Substack, Beehiiv, etc. |
| **Notifications** | GitHub, LinkedIn, social media, CI/CD, automated alerts |
| **Receipts** | Purchases, invoices, shipping confirmations, billing |
| **Personal** | Direct emails from known contacts (not work) |
| **Work** | Emails from work domain/colleagues (scileads.com, etc.) |
| **Calendar** | Meeting invites, RSVPs, event reminders |

If only a few categories have emails, only show non-empty ones. If one category has 0 emails, skip it entirely.

### Step 4: Summarize Each Category

For each category group, generate a concise summary:
- **1-2 sentences per category** overview
- **Bullet points** for individual emails: `[Sender] — [one-line summary]`
- Include actionable items: "Needs your signature", "Meeting at 3pm", "Payment due"
- Pull out links or key details where relevant

Keep summaries BRIEF. The whole brief should be readable in 30 seconds.

### Step 5: Format for Discord

Discord does NOT render markdown tables. Use this format:

```
🌅 **Morning Brief — Friday, July 11, 2026**

📰 **Newsletters** (3)
• Every — Dan Shipper on AI agent architectures and tool use
• TechCrunch — 3 startup funding rounds, 1 major acquisition
• React Digest — React 20 beta, new server components patterns

🔔 **Notifications** (5)
• GitHub — 2 CI failures on warframe-todo-tracker (commit message check)
• LinkedIn — 3 new connection requests, 1 message
• Fastmail — Storage at 85%, consider cleanup

📦 **Receipts** (1)
• Amazon — "Your order has shipped" arriving Sunday

💼 **Work** (2)
• Sarah Stevens — Q3 planning doc ready for review
• Eilis Crickard — Team lunch next Thursday?

─────────────────
Inbox: 2 urgent emails waiting. Say "triage my inbox" to sort them.
```

Format rules:
- Use emoji headers for categories (📰 📦 🔔 💼 👤 📅)
- Use `•` bullets (Discord renders them cleanly)
- Bold category names with count: `**Newsletters** (3)`
- Separator line: `─────────────────`
- Footer: action prompt if urgent emails exist
- Max ~1800 chars per message (Discord limit is 2000, leave buffer)

### Step 6: Deliver

If running as a cron job: the final response is auto-delivered to the configured Discord channel.

If running on-demand: present the brief in the current conversation.

After delivery, save to memory: `Iris: last brief — [current_ISO_timestamp]`

### Step 7: Offer Actions

End the brief with one or two relevant action prompts:
- "Say 'triage my inbox' to clean up new emails"
- "Say 'draft replies' to respond to the urgent ones"
- "Reply to this message to tell Iris about anything I missed"

## Discord Format Reference

✅ DO:
- Emoji section headers (they're universal and render everywhere)
- Bold for emphasis: `**text**`
- Bullet lists with `•`
- Short, scannable lines (one idea per bullet)
- Separator lines: `─────`

❌ DON'T:
- Markdown tables (Discord renders as raw text)
- Code blocks for non-code content (hard to read on mobile)
- Excessive emojis (one per section header is enough)
- Long paragraphs (nobody reads them in a brief)

## Pitfalls

- **Idempotency is critical**: Always track `Iris: last brief` timestamp and use `after:` filter. Never brief the same emails twice.
- **Discord 2000-char limit**: Briefs must stay under 2000 characters. If approaching limit, split into multiple messages or be more concise. Trim newsletter summaries aggressively.
- **Empty brief**: If no emails since last brief, still deliver: "🌅 **Morning Brief** — Nothing new since yesterday. Inbox zero! 🎉" Don't go silent — silence makes the user wonder if it's broken.
- **Timezone handling**: Use the timezone from `Iris delivery` memory for `after:` timestamps and for the date header in the brief. Europe/London means `+01:00` (BST) or `+00:00` (GMT) depending on date.
- **Archive folder**: Brief reads from Archive (where triage moved non-urgent emails). If the user hasn't run triage, Archive may be empty — include inbox emails too.
- **Memory update race**: If two briefs somehow run simultaneously, the last_brief timestamp could be wrong. The `after:` filter + archive-only reading minimizes this risk.

## Cron Job Setup

Two cron jobs to create (after the skill is committed):

**Morning Brief** (8:00 AM daily):
```
cronjob(
  action="create",
  name="Iris Morning Brief",
  schedule="0 8 * * *",
  prompt="Generate the morning email brief using the iris-brief skill. Summarize all emails since the last brief.",
  skills=["iris-brief"],
  deliver="discord:1515161336068444192",
  attach_to_session=true
)
```

**Afternoon Brief** (4:00 PM daily):
```
cronjob(
  action="create",
  name="Iris Afternoon Brief",
  schedule="0 16 * * *",
  prompt="Generate the afternoon email brief using the iris-brief skill. Summarize all emails since the morning brief.",
  skills=["iris-brief"],
  deliver="discord:1515161336068444192",
  attach_to_session=true
)
```

## Verification

After a brief run:
1. Brief delivered to correct Discord channel
2. Only emails since last brief are included (check timestamps)
3. Categories match `Iris categories` memory
4. Format is Discord-compatible (no tables, under 2000 chars)
5. `Iris: last brief` timestamp updated in memory
6. Morning brief at 8am, afternoon at 4pm (verify one fires)
