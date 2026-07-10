---
name: iris-draft
description: "Draft email replies in your voice. Reads emails needing reply, analyzes your writing style from sent mail, and creates drafts in your Drafts folder for review."
trigger: |
  User asks to "draft replies", "reply to emails", "respond to [name]",
  "write a draft", or after iris-inbox finds urgent emails needing response.
---

# Iris Draft Replies

Draft email replies in the user's writing voice. Analyzes the user's actual sent mail to clone their style, then creates draft replies in the Fastmail Drafts folder. **Never sends** — the user always reviews and sends manually.

## When to Use

- After inbox triage finds urgent emails needing reply
- User says "draft replies to my emails"
- User says "reply to [specific person]" or "respond to [email]"
- User wants to reply to an email but doesn't want to type it

## Safety Rules

1. **Never send** — use `draft_email` (not `send_email`). Drafts go to Drafts folder only.
2. **User always reviews** — explicitly tell the user to check Drafts folder before sending
3. **Voice must be grounded** — always sample sent mail for voice patterns, never guess
4. **One draft per thread** — don't create duplicate drafts for the same email thread

## Workflow

### Step 1: Load Voice Profile

Read `Iris voice` from Hermes memory. This contains the compact voice profile auto-detected during configuration.

Example: `Iris voice: warm-prof | Hi [Name], | Thanks, Namal | y`

Expand this into writing instructions:
- **Tone**: warm-professional → friendly but competent, personable, uses expressions like "thank you so much", "lovely to hear"
- **Greeting**: "Hi [Name]," (use recipient's first name)
- **Sign-off**: "Thanks,\nNamal"
- **Emoji**: yes, use sparingly in appropriate contexts (🙂, not excessive)
- **Style notes**: Medium sentences (15-25 words), mix of short and medium. Professional but approachable. Never stiff or overly formal.

### Step 2: Sample Recent Sent Mail (Voice Grounding)

To ground the voice profile with real examples, sample 3-5 recent sent emails:

```bash
search_email("from:me in:sent", limit=5)
```

Read the full body of 2-3 of them via `read_email`. Extract concrete examples of:
- How the user opens emails
- How they structure paragraphs
- How they sign off
- Their use of "!" or "thank you" or "hope you're well"
- How they handle different contexts (professional vs casual)

Use these real examples as reference when drafting — the draft should feel like it could have been written by the same person.

### Step 3: Find Emails Needing Reply

Option A: User specified a specific email/thread → use that directly.

Option B: User wants all urgent emails → search inbox for emails that need reply:
```
search_email("in:inbox is:unread")
```
Then filter to emails that:
- Are from a known contact (not noreply/notification/bulk)
- Contain a question or request
- Appear to expect a response

Option C: User says "reply to [name]" → search:
```
search_email("from:[name] in:inbox")
```

### Step 4: Read Target Emails

For each email needing reply, read the FULL body via `read_email(id, includeQuotedText=false)`. Understand:
- What the sender is asking/requesting
- The context and tone of their email
- Any specific information needed in the reply
- Whether it's professional, personal, or transactional

### Step 5: Draft the Reply

For each email, compose a draft reply. Follow these principles:

1. **Match the sender's context** — professional email gets professional reply, casual gets casual
2. **Use the user's voice patterns** — greeting, sign-off, tone, sentence style from Step 1 + 2
3. **Be helpful but concise** — answer questions directly, don't add fluff
4. **Acknowledge warmth** — if the sender was warm/friendly, mirror it (the user does this)
5. **Handle ambiguity honestly** — if information is missing, say "I'm not sure about X but..."

Create the draft via:
```
draft_email(
  mode="reply",
  emailId="[original_email_id]",
  body="[draft_body_in_markdown]"
)
```

### Step 6: Report Results

```
✍️ **Drafted [N] Replies**

📧 **To: [Sender Name]** — Re: [Subject]
Preview: [First line of draft...]
Status: Saved to Drafts folder ✓

📧 **To: [Sender Name]** — Re: [Subject]
Preview: [First line of draft...]
Status: Saved to Drafts folder ✓

Check your Drafts folder to review and send.
```

## Voice Profile Examples

Based on Namal's actual sent mail analysis:

**Professional context (SciLeads colleagues):**
```
Hi Eilis,

It was lovely meeting you and the rest of the team as well! Thank you so much for helping to organise everything and keeping in touch, it has been the best onboarding experience I've had to a new company.

I hope you have a great time in Australia!

Thanks,
Namal
```

**Scheduling context (colleague):**
```
Hi Cormac,

I'd be happy to meet at 9:30 am on Thursday, I'm flying in on Wednesday so that shouldn't be a problem. I'm not sure which hotel I've been booked in for yet but we can meet at the Europa either way.

Thanks,
Namal
```

**Key voice markers to replicate:**
- Opens with "Hi [Name]," (not "Dear")
- Uses "!" sparingly but genuinely (excitement, gratitude)
- "Thank you so much" for significant help
- "I hope you have a great [thing]" for warm send-offs
- Closes with "Thanks,\nNamal" — consistent sign-off
- Direct but warm — asks questions directly, offers alternatives
- Never stiff or corporate — personable even in professional contexts

## Pitfalls

- **Don't fabricate information**: If the email requires knowledge the user hasn't shared, draft a placeholder like "[Your answer about X]" and flag it for the user to fill in.
- **Don't over-emojify**: The user uses emojis sparingly (🙂, ☺️). Don't add 😊🎉✨ to every draft.
- **Don't be too casual**: Even though the user is warm, maintain appropriate professionalism with work contacts. Match the sender's tone.
- **Check for existing drafts**: Before creating a draft, check if one already exists for that thread to avoid duplicates.
- **Rate limiting**: Batch read_email calls, don't read 50 emails in sequence. Process 5-10 emails max per run.
- **Draft mode only**: Remember `draft_email` puts it in Drafts folder. The user must manually send from Fastmail. Never use any send capability.

## Verification

After running:
1. Drafts appear in Fastmail Drafts folder (check via `search_email("in:drafts")`)
2. Each draft is a reply to the correct thread (check threadId matches)
3. Voice matches the user's style (greeting, sign-off, tone)
4. Content addresses what the original email asked
