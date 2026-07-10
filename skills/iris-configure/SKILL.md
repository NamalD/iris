---
name: iris-configure
description: "Configure Iris email assistant preferences — voice style, importance rules, categories, and delivery settings. Use when setting up Iris for the first time or adjusting preferences."
trigger: |
  User mentions configuring Iris, setting up email assistant, changing Iris preferences,
  or asks about Iris voice/rules/categories/delivery settings.
---

# Iris Configure — Email Assistant Preferences

Set up and manage Iris preferences stored in Hermes memory. Iris reads these preferences every time it triages email, drafts replies, or generates briefs.

## When to Use

- First-time Iris setup
- User says "configure Iris", "set up my email assistant", "update my preferences"
- After correcting Iris behavior: "that email wasn't important" → update importance rules
- When changing delivery channels or schedule

## Memory Schema

Iris uses four compact memory entries (target: `memory`):

```
Iris voice: [tone] | [formality] | [sign-off] | [emoji: y/n] | [signature]
Iris important: [contacts] | [keywords] | [domains]
Iris categories: [cat1, cat2, ...]
Iris delivery: [discord_channel_id] | [tz] | [brief: morning/afternoon/both]
```

Each entry MUST be a single line under 300 chars. Use pipes as delimiters, keep values abbreviated.

## Workflow

### Step 1: Check Existing Config

First, check if Iris memory entries already exist. The memory tool will show them if present.

If none exist → first-time setup (Step 2).
If entries exist → show current config, ask if user wants to update (Step 3).

### Step 2: First-Time Setup

Walk the user through each preference area. Ask ONE question at a time using the `clarify` tool. Gather all answers, then save in one batch at the end.

**Voice Style** — How should Iris draft replies?

Iris auto-detects your writing voice by analyzing your recent sent emails. No manual description needed.

Voice detection process:
1. Search sent mail: `search_email("from:me in:sent", limit=10)`
2. Read 3-5 full email bodies via `read_email`
3. Extract patterns:
   - **Greeting**: "Hi [Name]," vs "Dear" vs none
   - **Sign-off**: "Thanks, Namal" vs "Best" vs "Cheers"
   - **Tone**: warm-professional / direct-minimal / formal / casual
   - **Emoji**: frequency and context (y/n, where used)
   - **Sentence style**: length, punctuation patterns, exclamation marks
   - **Politeness markers**: "Thank you so much", "Sorry to pester", etc.
4. Compile into compact voice profile

Store format: `Iris voice: [tone] | [greeting] | [sign-off] | [emoji: y/n]`

Example: `Iris voice: warm-prof | Hi [Name], | Thanks, Namal | y`

If sent mail is unavailable or insufficient, fall back to asking the user to describe their style with the clarify tool.

**Important People** — Whose emails should always stay in your inbox?

Question: "Who are the most important people whose emails should always get your attention? (names or email addresses, comma-separated)"

After getting the answer, store as: `Iris important: [contacts] | [keywords] | [domains]`

Example: `Iris important: boss@co.com, mom@pm.me | urgent,ASAP,deadline | namal.dev`

**Categories** — How should Iris group emails in briefs?

Question: "What categories should Iris use to group emails in your brief?"

Choices:
- "Default: Newsletters, Notifications, Receipts, Personal, Work, Calendar"
- "Simple: Newsletters, Receipts, Everything Else"
- "Let me customize"

Store format: `Iris categories: [cat1, cat2, ...]`

**Delivery** — Where should briefs go?

Question: "Where should Iris deliver your briefs, and what timezone are you in?"

Collect: Discord channel ID (use current chat), timezone, brief schedule preference.

Store format: `Iris delivery: [discord_channel_id] | [tz] | [brief_schedule]`

Example: `Iris delivery: 1515161336068444192 | Europe/London | both`

### Step 3: Update Existing Config

Show current preferences in a readable format. Ask what they want to change. Update only the changed entries.

Format for display:
```
🔧 **Iris Configuration**

**Voice**: friendly | casual | Best | emoji: yes
**Important**: boss@co.com, mom@pm.me | urgent, ASAP | namal.dev
**Categories**: Newsletters, Notifications, Receipts, Personal, Work
**Delivery**: Discord #home | Europe/London | both briefs
```

Then ask: "What would you like to change?" with choices:
- "Voice style"
- "Important people/rules"
- "Categories"
- "Delivery settings"
- "Everything looks good"

### Step 4: Save to Memory

After collecting all preferences, save to memory in ONE batch call. Use operations array for atomicity.

```python
# Pseudocode — use the memory tool with operations array
memory(target="memory", operations=[
    {"action": "add", "content": "Iris voice: friendly | casual | Best | y | Namal"},
    {"action": "add", "content": "Iris important: boss@co.com | urgent,ASAP | namal.dev"},
    {"action": "add", "content": "Iris categories: Newsletters, Notifications, Receipts, Personal, Work"},
    {"action": "add", "content": "Iris delivery: 1515161336068444192 | Europe/London | both"},
])
```

If replacing existing entries, use `old_text` with a unique substring from the old entry.

### Step 5: Confirm

After saving, display the saved configuration and confirm:

"Iris is configured! 🎉 Your preferences are saved. I'll use these when triaging your inbox, drafting replies, and generating briefs. Say 'triage my inbox' or 'generate my brief' whenever you're ready."

## Pitfalls

- **Memory size**: Hermes memory has ~2,200 chars. Iris entries must stay compact. If the combined entries exceed budget, shorten values (use initials, truncate lists). Never split one logical config across two entries.
- **Special characters**: Avoid `|` inside field values — pipes are delimiters. Use commas for internal lists instead.
- **Discord IDs**: The delivery channel ID must be a real Discord channel ID (snowflake). Use the current chat's channel ID by default.
- **Batch saves**: Always save all four entries in ONE memory() call with an operations array. Multiple calls risk partial saves if one fails.
- **On update**: Use `replace` action with `old_text` matching a substring of the existing entry — don't duplicate entries by adding alongside old ones.

## Verification

After configuring:
1. Check memory with `memory(target="memory")` — all four Iris entries should be present
2. Confirm format: each entry is a single line, pipe-delimited, under 300 chars
3. Verify the categories list matches what the user chose
4. Verify the delivery channel ID is a valid snowflake (18-19 digit number)
