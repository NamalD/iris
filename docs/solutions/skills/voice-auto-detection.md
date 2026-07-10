---
problem_type: pattern
category: skills
tags: [iris, voice, writing-style, auto-detection, fastmail]
date: 2026-07-10
issue: "#2"
---

# Voice Auto-Detection from Sent Mail

## Problem

The initial iris-configure skill asked the user to describe their writing voice manually ("Professional & concise" / "Friendly & warm" / "Direct & minimal"). The user rejected this approach, wanting Iris to clone their actual voice from real emails instead of asking them to self-describe.

## Root Cause

Self-reported writing style is unreliable. People don't accurately describe how they write — they describe how they *think* they write. The gap between self-perception and actual prose is large enough to produce drafts that don't feel like the user.

Spiral (writewithspiral.com) uses "stylometry" — the 200-year-old science of analyzing linguistic patterns — to build voice profiles. Iris can do the same with Fastmail MCP access to sent mail.

## Solution

**Instead of asking the user to describe their voice, auto-detect it from sent mail:**

1. Search sent mail: `search_email("from:me in:sent", limit=10)`
2. Read 3-5 full email bodies via `read_email`
3. Extract concrete patterns:
   - **Greeting**: How does the user open? "Hi [Name]," vs "Dear" vs no greeting
   - **Sign-off**: "Thanks,\nNamal" vs "Best" vs "Cheers"
   - **Tone**: warm-professional, direct-minimal, formal, casual
   - **Emoji usage**: frequency, context, which emojis
   - **Sentence style**: length, punctuation, exclamation marks
   - **Politeness markers**: "Thank you so much", "Sorry to pester", "I hope you have a great..."
4. Compile into compact voice profile for memory:
   ```
   Iris voice: warm-prof | Hi [Name], | Thanks, Namal | y
   ```
5. Include concrete email examples in the draft skill so the LLM has real reference material when composing replies

## Result

Namal's voice profile detected from 10 sent emails:
- Greeting: "Hi [Name],"
- Sign-off: "Thanks,\nNamal"  
- Tone: warm-professional (friendly but competent, uses "!" and "lovely", "thank you so much")
- Emoji: yes, sparingly (🙂)
- Style: personable even in professional contexts, never stiff or corporate

When drafting replies, iris-draft also reads 3-5 recent sent emails live to ground the LLM in real examples, not just the abstract profile.

## Gotcha: Voice Drift

If the user's writing style changes over time (new job, different context), the static memory profile becomes stale. Mitigation: the iris-draft skill reads fresh sent mail samples on every run, so it always has current examples. The memory profile serves as a fallback/quick reference, not the sole source.

## Prevention

- All Iris skills that generate text should ground themselves in real user data, not just preferences
- When the user asks to configure something, first check if it can be auto-detected from existing data before asking
- The configure skill's fallback question is still available if sent mail is insufficient (new account, no sent history)
