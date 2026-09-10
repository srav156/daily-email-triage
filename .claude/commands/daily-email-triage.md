---
description: Run the fetch -> triage -> (calendar check for scheduling requests) -> draft -> (block confirmed times on the calendar) pipeline over the last 24 hours of Gmail.
argument-hint: "[max_emails]"
---

Run the daily email triage workflow. If an argument is given (`$ARGUMENTS`), treat it as `max_emails` — the maximum number of emails to process in this run; if omitted, process all emails found.

Never use any tool that sends email, deletes anything, modifies/removes labels, or updates/deletes/responds to calendar events. The one mutating capability beyond drafts is scoped narrowly: `create_event`, used only by `scheduler-agent`, only to block a single confirmed meeting time on the user's own primary calendar, with no attendees and notifications forced off (`notificationLevel: "NONE"`) — never a real invite to anyone else. Everything else in this workflow is read/search tools plus `create_draft`.

Execute this exact sequence:

1. **Fetch.** Invoke the `fetch-agent` subagent to search Gmail with query `newer_than:1d` and retrieve the full content (sender, subject, thread ID, message ID, body) of every matching thread from the last 24 hours. If `max_emails` was given, tell the fetch agent to cap its results at that many emails. Wait for its structured output before continuing.

2. **Triage.** Pass the fetch agent's full structured list of emails to the `triage-agent` subagent verbatim. The triage agent has no Gmail tools — it only reasons over the data it's handed. Wait for its structured output: for each email, `needs_reply`, `urgency`, `suggested_tone`, `is_scheduling_request`, `requested_timeframe`, and a reason.

3. **Calendar check (only for scheduling requests).** From the triage agent's output, filter to emails where `needs_reply == true` AND `is_scheduling_request == true`. If there are any, invoke the `calendar-agent` subagent for each one (or once per distinct `requested_timeframe` if several scheduling emails share the same window, to avoid redundant calls), passing that email's `requested_timeframe` as the window to check — e.g. "sometime next month", "before Friday the 24th", or "unspecified" if the sender gave no hint (calendar-agent then defaults to the next 7 days). calendar-agent resolves that into concrete dates itself and checks working-hours availability; it is strictly read-only and never creates, updates, deletes, or responds to events. Collect each result to hand to draft-agent for the corresponding email(s). If there are no scheduling requests, skip this step entirely.

4. **Draft.** From the triage agent's output, filter to only the emails where `needs_reply == true`. For each of those, pass along the original email content (sender, subject, thread ID, message ID, body) plus the triage decision (urgency, suggested_tone, is_scheduling_request, requested_timeframe) to the `draft-agent` subagent. For any email where `is_scheduling_request == true`, also pass that email's matching open calendar slots from step 3 so draft-agent can propose real times instead of vague availability language. The draft agent writes a complete reply for each and saves it as a Gmail draft via `create_draft`. Wait for its structured output, which includes a `confirmed_slot` per draft — populated only when that reply firmly confirmed one exact meeting time (not when it merely offered options for the other person to pick from).

5. **Block confirmed times on the calendar (only when a slot was actually confirmed).** From draft-agent's output, filter to drafts with a non-null `confirmed_slot`. For each one, invoke the `scheduler-agent` subagent to create exactly one personal calendar event for that slot (summary, date, start/end time, time zone) — no attendees, `notificationLevel: "NONE"`. If no draft confirmed a specific slot, skip this step entirely.

6. **Report.** Summarize the run for the user:
   - How many emails were fetched (and whether results were capped)
   - How many were flagged `needs_reply = true`, broken down by urgency
   - How many of those were scheduling requests, and (if any) the window checked for each and a summary of the calendar availability found
   - How many drafts were successfully created (and any that failed, with reasons)
   - The subject and thread ID of each draft created, noting which ones included proposed or confirmed calendar times
   - How many calendar events were created by scheduler-agent, with their titles and times (and any that failed, with reasons)

Do not skip or reorder this sequence, except that step 3 (calendar check) is skipped when no email is flagged as a scheduling request, and step 5 (calendar block) is skipped when no draft confirmed a specific slot. Do not call any Gmail or Calendar tool directly yourself — always delegate through the five subagents so tool access stays scoped to what each agent is restricted to. Never let calendar-agent call a mutating tool (it isn't granted any), and never let scheduler-agent do anything except a single no-attendee `create_event` call per confirmed slot — no other agent should ever be given any calendar tool.
