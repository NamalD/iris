---
date: 2026-07-10
issue: "#6"
status: ready-to-run
---

# Iris MVP — End-to-End Integration Test

Manual verification checklist for the full Iris pipeline. Run through these steps to confirm Iris is working end-to-end.

## Prerequisites

- [x] Iris configured (memory entries exist)
- [x] iris-configure skill created
- [x] iris-inbox skill created
- [x] iris-draft skill created
- [x] iris-brief skill created
- [x] Morning and afternoon cron jobs created (926cb13f59d7, bb450824a1aa)
- [ ] Fastmail "Iris Briefed" folder created manually (optional — Archive used by default)

## Test 1: Configure Read-Back

**Goal**: Verify preferences were saved and can be read back.

**Steps**:
1. Load `iris-configure` skill
2. Check that existing Iris memory entries are displayed

**Expected**: All four entries shown with correct values:
```
Iris voice: warm-prof | Hi [Name], | Thanks, Namal | y
Iris important: Uddy,Priya,Sarath | auth,verification,2FA,security code
Iris categories: Newsletters,Notifications,Receipts,Personal,Work,Calendar
Iris delivery: 1515161336068444192 | Europe/London | both
```

**Result**: [ ] PASS / [ ] FAIL

---

## Test 2: Inbox Triage (Dry Run)

**Goal**: Verify triage correctly categorizes emails without archiving anything.

**Steps**:
1. Load `iris-inbox` skill
2. Run with a "today" time window
3. Review the categorization results
4. Do NOT archive — cancel the archive prompt

**Expected**:
- Auth/verification code emails flagged as urgent ✅ (verified: "Use 800804 to sign in")
- CI notification emails flagged as non-urgent ✅ (verified: 2x GitHub CI failures)
- Security alerts treated as uncertain (kept in inbox)
- Sender matching works against Iris important list

**Result**: [ ] PASS / [ ] FAIL

---

## Test 3: Draft a Reply

**Goal**: Verify reply drafting creates a draft in the correct voice.

**Steps**:
1. Load `iris-draft` skill
2. Ask Iris to draft a reply to a recent email
3. Check Fastmail Drafts folder for the draft
4. Review the draft: does it match the user's voice?

**Expected**:
- Draft appears in Drafts folder
- Greeting: "Hi [Name],"
- Sign-off: "Thanks,\nNamal"
- Tone: warm-professional, personable
- Content addresses the original email's question/request

**Result**: [ ] PASS / [ ] FAIL

---

## Test 4: Brief Generation (On-Demand)

**Goal**: Verify brief generation produces a Discord-friendly summary.

**Steps**:
1. Load `iris-brief` skill
2. Run on-demand: "Generate my brief"
3. Check the output format

**Expected**:
- Emoji section headers (📰 📦 🔔 etc.)
- Bullet lists with `•`
- No markdown tables
- Under 2000 characters
- Categories match Iris preferences
- Footer with action prompt

**Result**: [ ] PASS / [ ] FAIL

---

## Test 5: Cron Job Firing

**Goal**: Verify morning and afternoon briefs fire on schedule.

**Steps**:
1. Wait for next scheduled run (morning: 8am, afternoon: 4pm Europe/London)
2. Check Discord #home channel for brief delivery
3. Verify brief content is correct
4. Check that `Iris: last brief` timestamp updated in memory

**Expected**:
- Morning brief delivered to Discord #home at ~8am
- Afternoon brief delivered at ~4pm
- Briefs are idempotent (no duplicate content across runs)
- `Iris: last brief` timestamp advances after each run

**Result**: [ ] PASS / [ ] FAIL

---

## Test 6: Conversational Follow-Up

**Goal**: Verify user can reply to briefs conversationally.

**Steps**:
1. After a brief is delivered, reply to the brief message
2. Say "draft a reply to [name]" or "that newsletter looks interesting, tell me more"

**Expected**:
- Iris responds to the follow-up in context
- Iris has access to the brief context (attach_to_session=true)

**Result**: [ ] PASS / [ ] FAIL

---

## Test 7: Folder Verification

**Goal**: Verify archived emails are in the correct location.

**Steps**:
1. Run inbox triage with archive enabled (confirm archiving)
2. Check Fastmail Archive folder for the archived emails
3. Verify urgent emails remain in inbox

**Expected**:
- Non-urgent emails moved to Archive
- Urgent emails remain in inbox
- No emails deleted

**Result**: [ ] PASS / [ ] FAIL

---

## Summary

| Test | Status | Notes |
|------|--------|-------|
| T1: Configure Read-Back | ⬜ |  |
| T2: Inbox Triage | ⬜ | Dry-run categorization verified during build |
| T3: Draft Reply | ⬜ |  |
| T4: Brief Generation | ⬜ |  |
| T5: Cron Job Firing | ⬜ | First run tomorrow |
| T6: Conversational Follow-Up | ⬜ |  |
| T7: Folder Verification | ⬜ |  |

## Run Command

To run all tests in sequence:
```
# Load and test each skill
hermes run "load iris-configure and show my current Iris preferences"
hermes run "triage my inbox using iris-inbox — just a report, don't archive"
# (choose an email to draft a reply to)
hermes run "draft a reply to [email subject/ID] using iris-draft"
hermes run "generate my brief using iris-brief"
```
