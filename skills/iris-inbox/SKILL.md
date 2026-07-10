---
name: iris-inbox
description: "Triage your Fastmail inbox — categorize unread emails as urgent or non-urgent, archive non-urgent, and report results. Use when the user wants to clean their inbox or check what needs attention."
trigger: |
  User says "triage my inbox", "sort my email", "clean up inbox",
  "what's important", "clear the noise", or similar inbox management requests.
---

# Iris Inbox Triage

Read new unread emails from your inbox, categorize each as urgent (needs your attention/reply) or non-urgent (can be summarized in a brief later), and archive the non-urgent ones.

## When to Use

- User says "triage my inbox", "sort my email", "clean up"
- After configuring Iris for the first time
- Periodically to maintain inbox zero
- When the user feels overwhelmed by email

## Safety Rules

1. **Never delete emails** — only archive (move to Archive folder) or leave in inbox
2. **Forward-only** — only process emails received since the last triage (or today if first run). Never backfill historical email.
3. **Dry-run mode** — on first run or when unsure, categorize without archiving. Let the user review before acting.
4. **Batch limit** — process max 50 emails per run to avoid rate limiting
5. **Sender whitelist** — emails from "Iris important" senders stay in inbox always

## Workflow

### Step 1: Load Preferences

Read Iris preferences from Hermes memory:
- `Iris important` — who to never archive
- `Iris delivery` — for timezone-aware time window

### Step 2: Determine Time Window

Check memory for `Iris: last triage — [timestamp]`. 

If found: only process emails received after that timestamp.
If not found (first run): only process emails from today (use timezone from Iris delivery config).

Use `search_email` with date filter: `in:inbox is:unread after:[timestamp]`. Limit to 50 results.

### Step 3: Fetch and Categorize

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

### Step 4: Report Results

Before archiving anything, present the triage results:

```
📬 **Inbox Triage — [date]**

⚡ **Urgent** (staying in inbox): N emails
• [Sender] — [Subject preview]

📦 **To Archive**: N emails
• [Sender] — [Subject preview]

🤷 **Uncertain**: N emails (kept in inbox)
• [Sender] — [Subject preview]
```

Then ask: "Archive the [N] non-urgent emails?" with choices:
- "Yes, archive them"
- "No, leave everything as-is"
- "Let me review the list first"

### Step 5: Archive

Use `archive_email(ids=[...])` to archive all confirmed non-urgent emails in one call.

After archiving, save `Iris: last triage — [current_ISO_timestamp]` to memory.

### Step 6: Offer Next Steps

After triage:
- If urgent emails found: "Want me to draft replies to the [N] urgent emails? Load iris-draft."
- Report: "Triaged [N] emails: [X] urgent in inbox, [Y] archived. Inbox is [N] emails now."

## Pitfalls

- **Don't process 3K emails**: Fastmail inboxes can have thousands of unread. Always use time windows (today or since last triage). Never do `search_email("in:inbox is:unread")` without a date filter.
- **Archive vs Folder**: Iris uses Fastmail's Archive folder by default. For a separate "Iris Briefed" folder, the user must create it manually in Fastmail settings. The skill can be updated to use `update_email(ids=[...], folder="Iris Briefed")` once that folder exists.
- **Uncertain = keep**: When in doubt, leave the email in the inbox. False negatives (archiving something important) are worse than false positives (keeping a newsletter).
- **Sender matching**: Match `Iris important` contacts against both `name` and `email` fields. Use case-insensitive partial matching. "Uddy" should match "uthpala dayarathna" if that's in the important list.
- **Rate limiting**: Batch archive in one `archive_email` call with all IDs. Don't loop per-email.
- **Idempotency**: Track `last_triage_timestamp` to avoid reprocessing the same emails on consecutive runs.

## Edge Cases

- **No unread emails**: Report "Inbox zero! 🎉 Nothing to triage."
- **All urgent**: "All [N] unread emails look important — nothing to archive."
- **All non-urgent**: Archive everything, report "Inbox zero! Cleared [N] emails."
- **Mixed with many**: Group by urgency level for readability
- **First run with thousands**: The "today" time window prevents processing backlog

## Verification

After running:
1. Urgent emails remain in inbox
2. Archived emails moved to Archive folder (check `in:archive` search)
3. `Iris: last triage` timestamp updated in memory
4. No errors from rate limiting or invalid IDs
