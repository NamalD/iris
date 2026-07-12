---
name: iris-inbox
description: "Triage your Fastmail inbox — categorize unread emails as urgent or non-urgent, archive non-urgent, and report results. Also supports interactive backlog sweep mode for cleaning up historical emails in batches of 50. Use when the user wants to clean their inbox, triage email, or sweep old backlog."
trigger: |
  User says "triage my inbox", "sort my email", "clean up inbox",
  "what's important", "clear the noise", "sweep my inbox backlog",
  or similar inbox management requests.
---

# Iris Inbox Triage

Read unread emails from your inbox, categorize each as urgent (needs your attention/reply) or non-urgent (can be summarized in a brief later), and archive the non-urgent ones. Supports both forward-only triage and interactive historical backlog sweep.

## When to Use

- User says "triage my inbox", "sort my email", "clean up"
- After configuring Iris for the first time
- Periodically to maintain inbox zero
- When the user feels overwhelmed by email
- User says "sweep my inbox backlog" to clear historical emails in batches of 50

## Safety Rules

1. **Never delete emails** — only archive (move to Archive folder) or leave in inbox
2. **Forward-only by default** — normal triage only processes emails since the last triage (or today if first run)
3. **Dry-run mode** — on first run or when unsure, categorize without archiving. Let the user review before acting.
4. **Batch limit** — process max 50 emails per run to avoid rate limiting
5. **Sender whitelist** — emails from "Iris important" senders stay in inbox always

## Workflow

### Step 1: Determine Mode

Check the user's request:
- **Normal triage** ("triage my inbox"): forward-only, use time window since last triage or today
- **Backlog sweep** ("sweep my inbox backlog"): process historical emails in batches of 50, no lower date bound, oldest first

### Step 2: Load Preferences

Read Iris preferences from Hermes memory:
- `Iris important` — who to never archive
- `Iris delivery` — for timezone-aware time window

For backlog sweep, also check: `Iris: last sweep — [timestamp]`

If a last sweep timestamp exists, only process emails received after that timestamp. If not found, start from the beginning of the inbox (oldest first).

### Step 3: Fetch Emails

**Normal triage:**
Use `search_email` with time filter: `in:inbox is:unread after:[last_triage_timestamp]`. Limit to 50 results.

**Backlog sweep:**
Use `search_email` with: `in:inbox after:[last_sweep_timestamp]`. Limit to 50 results.
- If no `last_sweep_timestamp`: use `in:inbox` with no `after:` filter, limit 50, oldest first
- Sort by receivedAt ascending for sweep mode (oldest first)

### Step 4: Categorize

For each email, read `subject`, `preview`, `from` (name + email). Do NOT read full bodies — subject+preview is enough for categorization in most cases.

For each email, determine category:

**URGENT** (stays in inbox) if ANY of:
- Sender matches an "Iris important" contact (name or email partial match)
- Subject contains: urgent, ASAP, action required, deadline, time-sensitive
- Email appears to be a direct personal message (not bulk/newsletter/notification)
- Email contains a question or request directed at the user
- Auth/verification/security codes (matches "auth, verification, 2FA, security code" keywords)
- Sender is a known contact AND the email is personally addressed

**NON-URGENT** (can be archived) if it's clearly:
- A newsletter (bulk sender, unsubscribe link pattern)
- A notification (GitHub, LinkedIn, social media, calendar reminders)
- A receipt or order confirmation (purchase, shipping, billing)
- A promotional/marketing email
- An automated/system message that doesn't require action
- A calendar invite (can be seen in calendar; no email action needed)

**UNCERTAIN** — if the LLM cannot confidently categorize, keep in inbox. Default to keeping, not archiving.

### Step 5: Report Results

Before archiving anything, present the triage results:

**Normal triage format:**
```
📬 **Inbox Triage — [date]**

⚡ **Urgent** (staying in inbox): N emails
• [Sender] — [Subject preview]

📦 **To Archive**: N emails
• [Sender] — [Subject preview]

🤷 **Uncertain**: N emails (kept in inbox)
• [Sender] — [Subject preview]
```

**Backlog sweep format:**
```
🧹 **Backlog Sweep — Batch N**

⚡ **Urgent** (staying in inbox): N emails
• [Sender] — [Subject preview]

📦 **Archived**: N emails
• [Sender] — [Subject preview]

🤷 **Uncertain**: N emails (kept in inbox)
• [Sender] — [Subject preview]

─────────────────
Inbox: [remaining count] emails left
```

Then ask: "Archive the [N] non-urgent emails?" with choices:
- "Yes, archive them"
- "No, leave everything as-is"
- "Let me review the list first"

### Step 6: Archive

Use `archive_email(ids=[...])` to archive all confirmed non-urgent emails in one call.

After archiving:
- **Normal triage:** save `Iris: last triage — [current_ISO_timestamp]` to memory
- **Backlog sweep:** save `Iris: last sweep — [current_ISO_timestamp]` to memory

### Step 7: Offer Next Steps

**Normal triage:**
- If urgent emails found: "Want me to draft replies to the [N] urgent emails? Load iris-draft."
- Report: "Triaged [N] emails: [X] urgent in inbox, [Y] archived. Inbox is [N] emails now."

**Backlog sweep:**
- Report: "Swept batch N: [X] archived, [Y] urgent, [Z] uncertain. [N] emails remaining in inbox."
- Offer: "Run another sweep batch?" or "Say 'triage my inbox' to process new mail going forward."

## Pitfalls

- **Don't process 3K emails at once**: Always use batch limit of 50 per run. Never process the full inbox in one go.
- **Backlog sweep is oldest-first**: Use `after:` timestamp or no date filter with ascending receivedAt sort. Don't accidentally process newest first and never catch up.
- **Archive vs Folder**: Iris uses Fastmail's Archive folder by default. For a separate "Iris Briefed" folder, the user must create it manually in Fastmail settings. The skill can be updated to use `update_email(ids=[...], folder="Iris Briefed")` once that folder exists.
- **Uncertain = keep**: When in doubt, leave the email in the inbox. False negatives (archiving something important) are worse than false positives (keeping a newsletter).
- **Sender matching**: Match `Iris important` contacts against both `name` and `email` fields. Use case-insensitive partial matching. "Uddy" should match "uthpala dayarathna" if that's in the important list.
- **Rate limiting**: Batch archive in one `archive_email` call with all IDs. Don't loop per-email.
- **Idempotency**: Track `last_triage_timestamp` for normal triage, `last_sweep_timestamp` for backlog sweep, to avoid reprocessing the same emails.
- **Sweep mode doesn't replace normal triage**: After sweeping the backlog, the user should run normal triage to process new mail. The sweep clears history; triage handles the present.

## Edge Cases

- **No unread emails**: Report "Inbox zero! 🎉 Nothing to triage."
- **All urgent**: "All [N] unread emails look important — nothing to archive."
- **All non-urgent**: Archive everything, report "Inbox zero! Cleared [N] emails."
- **Mixed with many**: Group by urgency level for readability
- **Sweep with thousands remaining**: Report remaining count after each batch so user knows progress
- **Sweep finds mostly urgent**: If a batch has 40+ urgent emails, warn the user and suggest they update their `Iris important` list

## Verification

After running:
1. Urgent emails remain in inbox
2. Archived emails moved to Archive folder (check `in:archive` search)
3. `Iris: last triage` or `Iris: last sweep` timestamp updated in memory
4. No errors from rate limiting or invalid IDs
5. Inbox count decreased by number of archived emails
