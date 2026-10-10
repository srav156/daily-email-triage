# Daily Email Triage — Multi-Agent Claude Code Workflow

A multi-agent, multi-skill email workflow built entirely as Claude Code subagents and a slash command — no standalone application code. It triages your Gmail inbox, drafts replies for anything that actually needs one, checks your real calendar availability for scheduling requests, and — only when a specific time gets firmly confirmed in a reply — blocks that time on your calendar. It never sends, deletes, or modifies anything that already exists; the only mutating actions anywhere in this workflow are creating a Gmail draft and creating a private calendar hold.

There's also a separate Python port of this same workflow (`Gmail-Multi-Agent`, not included in this repo) that trades this version's zero-setup live simplicity for a real, testable, deterministic-by-default code structure. This project is the original. In its base setup it runs live today inside a Claude Code session, with no dependencies to install and no API key to manage. It has also been run outside Claude Code, in a persistent agent (Hermes) and in n8n; those setups need extra configuration, such as an Anthropic API key for Hermes. See [Deployment phases](#deployment-phases).

## What "agent" means here

Every "agent" in this project is the same underlying model — Claude — launched fresh each time as a different persona: a markdown file with YAML frontmatter (`name`, `description`, `tools`) plus a system prompt. The `tools:` field is what actually restricts what each agent can do; it's an allowlist enforced by the Claude Code harness, not just a suggestion in the prompt. There's no separate AI system or code layer behind any of this — the multi-agent structure is "one model, invoked multiple times, each time with a different tool allowlist and a different prompt," orchestrated by a Claude Code session reading the command file below and calling each subagent in sequence.

## The five agents (`.claude/agents/`)

- **`fetch-agent.md`** — searches Gmail (`newer_than:1d`) and retrieves full content (sender, subject, thread ID, message ID, body) for matching threads. Tools: `search_threads`, `get_thread`, `get_message`, `list_drafts`. Read-only.
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
- Only Gmail search/get-message, list-drafts (read-only) and create-draft are ever used by any agent — no delete, no label-modify.
- calendar-agent is strictly read-only. It is never granted `create_event`, `update_event`, `delete_event`, or `respond_to_event`.
- scheduler-agent is the **one** agent with write access to the calendar, and only to `create_event`. It's instructed to never add attendees and to always set `notificationLevel: "NONE"` explicitly — this is enforced by tool-allowlist (which tool it can call) plus prompt instructions (how it calls it), since there's no code layer between an agent and the MCP tool call in this project. See the Python port's `README.md` for how that same guarantee becomes structural (a Python function signature) rather than instructed, in a `.py` port with real code.
- Every agent's tool access is scoped by the harness via each file's `tools:` frontmatter field — an agent literally cannot call a tool it isn't listed for, regardless of what it decides to do.

## Running it

```
claude -p 'daily-email-triage'
claude -p 'daily-email-triage 5'   # cap at 5 emails
```

Requires the "claude.ai Gmail" and "claude.ai Google Calendar" MCP connectors already authorized for your account (OAuth-based, one-time setup). For this local Claude Code setup there is no separate API key, no `pip install`, and nothing to configure beyond that. Running it elsewhere (Hermes, n8n) needs more setup; see [Deployment phases](#deployment-phases).

## Deployment phases

Beyond running locally in Claude Code, this pipeline was also run in two other environments. The core constraints stayed the same in both: Gmail is draft-only and nothing is ever sent.

### Phase 1: Hermes Agent + Telegram

Hermes Agent (an open-source agent by Nous Research) is a separate program from Claude Code. It runs persistently and can push messages on its own, which Claude Code can't do unattended.

- **Backend model:** Hermes used a Claude model through an Anthropic API key from [console.anthropic.com](https://console.anthropic.com). An Anthropic OAuth connection conflicted with the API key and was removed.
- **Telegram gateway:** a bot token is added to the Hermes config. Cron output also needs a **Home Channel ID** (your Telegram user ID). Without it, delivery fails silently.
- **Gmail and Calendar:** Hermes can't use the Claude.ai-hosted connectors, so it needs its own Gmail and Calendar MCP authorization.
- **Agent definitions:** Hermes doesn't read `.claude/agents/*.md`, so the agents were recreated in Hermes's own format with the same constraints (draft-only Gmail, no sending).
- **Result:** scheduled triage with the summary delivered to Telegram, plus a confirmed two-way conversational flow.

### Phase 2: Self-hosted n8n + Slack

- **Container:** n8n ran in Docker on Apple Silicon, with a persistent volume. A custom image (n8n base plus the Claude Code CLI) mounts the `.claude` folder into the container.
- **Public HTTPS:** ngrok gives n8n a public HTTPS domain. This is required because Slack OAuth rejects `localhost` redirect URIs.
- **Credentials:** Google OAuth2 (Gmail, Calendar, Sheets, Drive, Docs), Slack and Notion.
- **Workflow:** Slack Trigger → Filter (trigger phrase) → Code node running `claude -p "/daily-email-triage"` → Slack node posting the triage report back to the channel.
- **Schedule trigger:** an optional Schedule Trigger (every 15 minutes) was added in parallel, then deactivated.

#### Problems solved

- **Headless MCP failure.** `claude -p` couldn't reach the Claude.ai Gmail and Calendar connectors when `CLAUDE_CODE_OAUTH_TOKEN` was set, because that variable forces a scope-limited auth mode. The fix is to unset it for the call:
  ```
  env -u CLAUDE_CODE_OAUTH_TOKEN claude -p "/daily-email-triage"
  ```
  This uses the interactive OAuth login stored in the mounted `~/.claude`. A live run confirmed it: 21 real emails fetched and triaged, and a real draft created.
- **Slack Socket Mode** silently overrides the Event Subscriptions Request URL. Disable it.
- **Missing Execute Command node.** It wasn't available in this n8n build. A Code node using `child_process.exec` replaced it.
- **Code node sandbox.** It blocked `child_process`. Setting `NODE_FUNCTION_ALLOW_BUILTIN=child_process` on the container fixed it.
- **Slack Trigger only fires when the workflow is Published.** There is a test vs. production webhook mismatch, so test with real Slack messages rather than "Execute step".
- **Two Slack credentials.** The Slack Trigger needs a "Slack API" credential (bot token plus signing secret), separate from the OAuth2 credential used by the Slack action node.

#### Safety notes

- Draft-only: the pipeline never sends email.
- Never commit `CLAUDE_CODE_OAUTH_TOKEN`, API keys, bot tokens or your ngrok domain. Use placeholders in anything you publish.

## Duplicate-draft protection

Gmail search excludes drafts, so a thread that only has a draft reply would look unanswered on a re-run. To prevent duplicate drafts, fetch-agent also has read-only `list_drafts` access: once per run it collects the thread IDs that already have a draft and tags each email with `has_existing_draft`. The command then skips those emails for the calendar, draft, and calendar-block steps, and draft-agent refuses them as a backstop. Skipped threads are listed in the final report.

Limitation: detection is per thread, so if you delete or send the draft the thread is eligible again, and a draft you wrote yourself also suppresses a new one.

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
