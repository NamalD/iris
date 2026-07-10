# Feature Gap Analysis: Cora vs Iris (Hermes Agent)

Date: 2026-07-10
Status: Complete — ready for planning phase

## Summary

Cora is a Gmail-only, web-based AI email assistant that costs $20-39/month. Iris will be a Fastmail-based, Hermes Agent-powered alternative that's free and self-hosted. This document maps every Cora feature to what Hermes can do today, identifying gaps and opportunities.

## Cora Feature Map

### Core Email Processing

| Cora Feature | Hermes Capability | Status | Notes |
|---|---|---|---|
| **Inbox screening** — determines what's urgent vs non-urgent | `search_email` + LLM categorization | ✅ Feasible | LLM reads email subject+preview, classifies based on user rules |
| **Auto-archive non-urgent** — moves non-urgent out of inbox | `archive_email` / `update_email` to folder | ✅ Feasible | Archive to "Iris Briefed" folder instead of Gmail's archive |
| **Urgent in inbox** — only emails needing a reply stay visible | Pass-through (Iris doesn't touch urgent) | ✅ Feasible | Default behavior — if Iris doesn't archive it, it stays in inbox |
| **Time-sensitive detection** — knows what's personally important | LLM reasoning on sender, subject, content | ✅ Feasible | Can use contact importance, keyword matching, user preferences |
| **Voice analysis** — learns writing style from email history | `search_email` + `read_email` on sent mail | ✅ Feasible | Sample recent sent emails, extract tone/style patterns with LLM |

### Drafting

| Cora Feature | Hermes Capability | Status | Notes |
|---|---|---|---|
| **Auto-draft replies** — drafts in user's voice for common requests | `draft_email` with mode='reply' | ✅ Feasible | LLM writes draft body, MCP creates draft in Drafts folder |
| **Multiple draft options** — provides 2-3 response variants | Multiple `draft_email` calls | ✅ Feasible | Can generate 2-3 variants per email |
| **Context-aware details** — inserts Calendly links, cc's assistant | LLM + user preferences in memory | ✅ Feasible | Store common patterns (links, cc patterns) in memory |
| **Never sends autonomously** — drafts only, user approves | `draft_email` never sends | ✅ Built-in | MCP `draft_email` goes to Drafts folder, never sent |
| **Repetitive email detection** — identifies common request patterns | LLM pattern matching across emails | ✅ Feasible | Compare subject/body similarity across recent emails |

### Brief

| Cora Feature | Hermes Capability | Status | Notes |
|---|---|---|---|
| **Twice-daily brief** — morning + afternoon summary | Hermes `cronjob` (recurring) | ✅ Feasible | Cron job at 8am + 4pm, runs iris-brief skill |
| **Beautiful formatted summary** — scannable, categorised | LLM-generated markdown → Discord/Telegram | ⚠️ Partial | Discord doesn't render tables. Use code blocks, bullet lists, emoji sections |
| **Calendar readout** — meetings, RSVPs, invites | `search_events` + summary | ✅ Feasible | Fastmail MCP has full calendar access |
| **Payment summaries** — spending at a glance | Email content parsing | ⚠️ Partial | Can detect payment confirmation emails, but no structured transaction parsing |
| **Newsletter summaries** — pulls interesting stories | LLM summarization of newsletter content | ✅ Feasible | Read newsletter emails, extract key stories/links |
| **Action items** — calls out things needing response | LLM identifies asks/questions in emails | ✅ Feasible | Key capability of LLM summarization |
| **"Add to-do"** — turns email items into tasks | `create_note` in Fastmail notes | ✅ Feasible | Can create a note/todo from brief items |

### Configuration & Customization

| Cora Feature | Hermes Capability | Status | Notes |
|---|---|---|---|
| **Plain-language rules** — prompt-based categorization | LLM follows user instructions in memory | ✅ Feasible | Store rules as Hermes memory or skill config |
| **Category overrides** — re-categorize misclassified emails | Conversational: "that's not a newsletter, it's important" | ✅ Feasible | User corrects Iris in chat, Iris updates memory |
| **Per-category brief sections** — newsletters, payments, etc. | LLM grouping in brief output | ✅ Feasible | Define categories in config, LLM sorts emails into sections |
| **Feature toggles** — turn drafts/brief on/off | Cron job pause/resume | ✅ Feasible | Hermes cronjob pause/resume |

### Interface

| Cora Feature | Hermes Capability | Status | Notes |
|---|---|---|---|
| **Web UI** — beautiful web app | N/A — Discord/Telegram delivery | 🔄 Different | Iris delivers via chat platforms, not a web app |
| **Mobile web** — works on phone browser | Discord/Telegram mobile apps | 🔄 Different | Native mobile apps via chat platforms |
| **Gmail integration** — works alongside Gmail | Fastmail integration | 🔄 Different | Fastmail instead of Gmail — same concept, different provider |
| **Chat with Cora** — conversational adjustments | Hermes conversation | ✅ Superior | Iris IS the conversation — every interaction is a chat |
| **"Next Brief" label** — review what was briefed | "Iris Briefed" folder | ✅ Feasible | Fastmail folder for all briefed emails |

### Cora Missing / Differentiators

| Feature | Hermes Status | Notes |
|---|---|---|
| **Multi-platform** (Discord + Telegram) | ✅ Unique | Cora is web-only. Iris delivers wherever you are |
| **Calendar integration** | ✅ Unique | Fastmail MCP has deeper calendar access than Gmail API |
| **Notes integration** | ✅ Unique | Can create Fastmail notes from emails |
| **Contact management** | ✅ Unique | Full Fastmail contacts CRUD |
| **Email sending** (when user asks) | ✅ Unique | Cora never sends. Iris can send if user explicitly asks |
| **Multi-account** (multiple Fastmail identities) | ✅ Unique | Fastmail supports multiple identities natively |
| **Historical email processing** | ⚠️ Not MVP | Cora is forward-only too; Iris could backfill as a future feature |

## Irreducible Gaps

| Gap | Severity | Explanation |
|---|---|---|
| **Beautiful web UI** | Low | Deliberate trade-off. Iris lives in chat (Discord/Telegram), not a standalone app. The brief is markdown, not a designed webpage. |
| **Payment parsing** | Medium | Cora has structured payment extraction. Iris can detect payment emails and include them in briefs, but won't parse amounts/dates/marchants with high accuracy without dedicated parsing logic. |
| **Calendar RSVP actions** | Low | Cora shows calendar with action buttons (RSVP). Iris can summarize calendar and RSVP via `rsvp_event` — just not with inline buttons in the brief. |

## Opportunity: What Iris Can Do That Cora Can't

1. **Multi-platform delivery** — brief arrives in Discord and Telegram, not another app to check
2. **Conversational email management** — "draft a reply to Sarah", "what did Amazon ship?", "archive all LinkedIn notifications"
3. **Calendar + Email unified** — same interface for email brief, calendar summary, RSVPs
4. **Note-taking from emails** — "create a note from this receipt" → Fastmail note
5. **Contact management** — update contacts from email signatures
6. **Custom automations** — "if I get an email from X, ping me on Discord immediately"
7. **Zero recurring cost** — runs on existing Hermes infra, no subscription
8. **Open source** — auditable, extensible, no vendor lock-in

## Verdict

**Iris can replicate 80%+ of Cora's core value** (inbox screening, drafts, briefs) with Hermes Agent + Fastmail MCP. The remaining 20% is primarily the web UI (which Iris replaces with chat delivery) and structured payment parsing (future enhancement). Iris also adds unique capabilities Cora lacks: multi-platform delivery, calendar integration, and conversational email management.
