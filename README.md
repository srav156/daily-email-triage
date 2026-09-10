# Daily Email Triage — Multi-Agent Claude Code Workflow

A multi-agent, multi-skill email workflow built entirely as Claude Code subagents and a slash command — no standalone application code. It triages your Gmail inbox, drafts replies for anything that actually needs one, checks your real calendar availability for scheduling requests, and — only when a specific time gets firmly confirmed in a reply — blocks that time on your calendar. It never sends, deletes, or modifies anything that already exists; the only mutating actions anywhere in this workflow are creating a Gmail draft and creating a private calendar hold.

There's a [Python port of this same workflow](../Gmail-Multi-Agent) that trades this version's zero-setup live simplicity for a real, testable, deterministic-by-default code structure. This project is the original: it runs live today, with no dependencies to install and no API key to manage, entirely inside a Claude Code session.

## What "agent" means here

Every "agent" in this project is the same underlying model — Claude — launched fresh each time as a different persona: a markdown file with YAML frontmatter (`name`, `description`, `tools`) plus a system prompt. The `tools:` field is what actually restricts what each agent can do; it's an allowlist enforced by the Claude Code harness, not just a suggestion in the prompt. There's no separate AI system or code layer behind any of this — the multi-agent structure is "one model, invoked multiple times, each time with a different tool allowlist and a different prompt," orchestrated by a Claude Code session reading the command file below and calling each subagent in sequence.

## The five agents (`.claude/agents/`)

- **`fetch-agent.md`** — searches Gmail (`newer_than:1d`) and retrieves full content (sender, subject, thread ID, message ID, body) for matching threads. Tools: `search_threads`, `get_thread`, `get_message`. Read-only.
- **`triage-agent.md`** — decides, per email, whether it needs a reply, its urgency, tone, and whether it's a scheduling request (plus the sender's own timeframe language, if any). No tools — pure reasoning over the data it's handed.
- **`calendar-agent.md`** — for emails flagged as scheduling requests, resolves the requested timeframe into a concrete window and checks real availability. Tools: `list_calendars`, `list_events`, `search_events`, `suggest_time`, `get_event` — all read-only. Never creates, updates, deletes, or responds to events.
- **`draft-agent.md`** — writes a complete reply for every email that needs one and saves it as a Gmail draft. For scheduling requests, proposes real open slots or firmly confirms a sender-proposed time that was verified free. Tool: `create_draft` only.
- **`scheduler-agent.md`** — the one write path onto the calendar. Fires only for a draft that firmly confirmed one exact time; creates exactly one personal calendar hold, with no attendees and notifications forced off. Tool: `create_event` only.

## Orchestration (`.claude/commands/daily-email-triage.md`)

The slash command `/daily-email-triage [max_emails]` describes the exact sequence a live session follows:

1. **Fetch** — run fetch-agent for the last 24 hours (capped, if an argument was given).
2. **Triage** — pass fetch-agent's output verbatim to triage-agent.
3. **Calendar check** *(only if any email is a scheduling request)* — run calendar-agent per distinct requested timeframe.
4. **Draft** — pass every `needs_reply=true` email, plus any calendar availability found, to draft-agent.
5. **Block confirmed times** *(only if a draft firmly confirmed a slot)* — run scheduler-agent for each confirmed slot.
6. **Report** — summarize counts, subjects, thread IDs, and what was actually created.

Steps 3 and 5 are the only conditional ones — a day of pure newsletters touches zero calendar tools.

## Non-negotiable constraints

- No "send email" tool exists in the connected Gmail MCP server at all — that half of the constraint is structural, not just a rule being followed.
- Only Gmail search/get-message and create-draft are ever used by any agent — no delete, no label-modify.
- calendar-agent is strictly read-only. It is never granted `create_event`, `update_event`, `delete_event`, or `respond_to_event`.
- scheduler-agent is the **one** agent with write access to the calendar, and only to `create_event`. It's instructed to never add attendees and to always set `notificationLevel: "NONE"` explicitly — this is enforced by tool-allowlist (which tool it can call) plus prompt instructions (how it calls it), since there's no code layer between an agent and the MCP tool call in this project. See the Python port's `README.md` for how that same guarantee becomes structural (a Python function signature) rather than instructed, in a `.py` port with real code.
- Every agent's tool access is scoped by the harness via each file's `tools:` frontmatter field — an agent literally cannot call a tool it isn't listed for, regardless of what it decides to do.

## Running it

```
claude -p 'daily-email-triage'
claude -p 'daily-email-triage 5'   # cap at 5 emails
```

Requires the "claude.ai Gmail" and "claude.ai Google Calendar" MCP connectors already authorized for your account (OAuth-based, one-time setup) — no separate API key, no `pip install`, nothing to configure beyond that.

## Known limitation

The pipeline has no way to see its own prior drafts — Gmail search excludes drafts by default — so re-running against a thread that only has a draft reply (not a sent one) will re-flag it and create a duplicate draft. Fixing this would mean giving fetch-agent or triage-agent read access to `list_drafts` (still read-only, fits the existing tool-scoping model) to check for an existing draft on a thread before flagging it again. Not yet implemented.

## Project layout

```
.claude/
├── agents/
│   ├── fetch-agent.md
│   ├── triage-agent.md
│   ├── calendar-agent.md
│   ├── draft-agent.md
│   └── scheduler-agent.md
└── commands/
    └── daily-email-triage.md
```
