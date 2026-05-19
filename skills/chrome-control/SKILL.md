---
name: chrome-control
description: Drive the user's real signed-in Chrome via the chrome-devtools MCP on CDP port 9222. Use whenever a task requires interacting with a logged-in web app the agent cannot reach with a normal HTTP fetch (dashboards, SaaS surfaces, file uploads, content behind auth). Do NOT use for headless scraping of public sites — use a normal fetch for those.
---

# chrome-control

Orchestrator guidance for browser tasks. **You do not call the chrome-devtools MCP tools yourself** (`mcp__chrome-devtools__*` in Claude Code, `chrome-devtools_*` in OpenCode). Delegate every browser interaction to the `chrome-subagent`. Its context is discarded on return, so raw snapshots, screenshots, and intermediate state never enter the parent transcript.

## When to delegate

Any task that requires the user's signed-in browser — read a value off a SaaS dashboard, click a button in a web app, download an attachment, scrape transcript text — goes to `chrome-subagent` via the host's subagent dispatcher (the `Agent` tool in Claude Code, the `Task` tool with `subagent_type: chrome-subagent` in OpenCode). The subagent owns the CDP connection and all per-action safety checks.

## Browser launch is NOT a subagent's job

Subagents must only **connect to an already-running Chrome on CDP 9222**. They must not launch, relaunch, or kill the browser. If Chrome is not up, the subagent's job is to fail fast and report it — the parent (or user) handles the launch.

**Why:** Subagents have no view of each other. When two run in parallel and Chrome isn't already up, each independently tries to launch it, and the window visibly opens/closes/opens as they fight. It also risks clobbering an existing session with the user's tabs.

**How to apply:**
- Before spawning any chrome-subagent, confirm Chrome is up on CDP 9222 (e.g. `lsof -i :9222`). If it isn't, stop and ask the user to launch it.
- Never run multiple chrome-subagents in parallel — serialize them against the single shared Chrome instance.
- In subagent prompts, say "connect to the existing Chrome on CDP 9222; do not launch", not "open Chrome and…".

## Decomposition rules

The subagent has no memory of this conversation, and its own context can flood if you cram too much into one call. Decompose accordingly:

1. **One sub-task per call.** "Open the page, read the value, then submit the form" is three calls, not one. Each call returns ≤3 lines; you stitch them together.
2. **Self-contained prompts.** Include URL or which tab, the exact data to write, the verification criterion, and the return shape. The subagent cannot ask you a clarifying question — if your prompt is ambiguous it will guess or stop.
3. **Verify with values, not images.** Ask for `status="confirmed"`, not "screenshot of the status pill". Screenshots are the subagent's internal tool; they should not come back to you.

## 4-line delegation template

```
Page: <url or "the open tab on app X">
Action: <one concrete operation>
Verify: <one DOM value to read back>
Return: <one-line outcome, ≤3 lines total>
```

Example:

```
Page: the open tab on demo-sheet at https://example.com/sheet/abc123
Action: set the value of the cell with data-cell-id="A89" to 88
Verify: read back data-cell-id="A89" via evaluate_script
Return: "Done. A89=<value>" or "Stopped. <reason>"
```

## Risk hints in the prompt

The subagent applies its own LOW/MED/HIGH risk tiering, but you have context it lacks. Flag what you know:

- "Personal scratch document, no co-editors" — invites LOW.
- "Document shared with a team — assume co-editors" — invites HIGH.
- "Form submission, will Submit at the end" — HIGH by definition; the Submit is irreversible.

Don't dictate the tier; describe the audience and reversibility, let the subagent classify.

## Caveats (still relevant — the subagent can hit them too)

- **Profile lock**: the dedicated `--user-data-dir` (e.g. `~/.chrome-canary-cdp-profile`) must be killed before relaunch or the new launch silently exits.
- **Chrome v136+ port block**: `--remote-debugging-port` is silently ignored on the default profile; the dedicated `--user-data-dir` bypasses it.
- **Port conflict**: if `lsof -i :9222` shows a non-Chrome listener, the subagent will fall back to 9223 and report it.

## Direct-drive escape hatch

For interactive debugging where you need to watch each step, you may call the chrome-devtools MCP tools directly (`mcp__chrome-devtools__*` / `chrome-devtools_*`). The chrome-subagent definition lives at `~/.claude/agents/chrome-subagent.md` for Claude Code and `~/.config/opencode/agents/chrome-subagent.md` for OpenCode. This path forfeits the context hygiene the subagent provides; use it sparingly.

## Cleanup

Chrome keeps running until killed. Leave it — next session reuses it.
