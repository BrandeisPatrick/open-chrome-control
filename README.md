# open-chrome-control

A Claude Code / OpenCode pattern for safely driving the user's real, signed-in Chrome via the [`chrome-devtools`](https://github.com/ChromeDevTools/chrome-devtools-mcp) MCP server.

The pattern splits browser work into two cooperating components:

1. **`chrome-control`** — an orchestrator skill loaded into the parent agent. It decides *what* the browser should do and decomposes the work, but never calls the chrome-devtools MCP itself.
2. **`chrome-subagent`** — a sub-agent with exclusive access to the chrome-devtools tools. It executes one concrete sub-task at a time, applies a risk-tiered safety model, and returns ≤3 lines of outcome text. Its context (snapshots, screenshots, intermediate DOM reads) is discarded on return.

The result: the parent transcript stays clean, browser state never pollutes long conversations, and irreversible actions pass through explicit safety gates.

## Why two components

A naive "let the parent agent click around" loop has two failure modes:

- **Context flooding.** A single accessibility snapshot can be 8000+ tokens. After three pages, the parent's context is mostly stale DOM and the model loses the original task.
- **Blind action.** Without a verification step before every click/type, the model acts on guesses. On reversible surfaces this is fine; on a Send / Submit / Delete button it isn't.

Splitting the orchestrator from the executor solves both: the executor can take all the snapshots and screenshots it needs to verify itself, then throw them away, returning only the outcome the orchestrator actually needs.

## Prerequisites

- **Chrome (Canary recommended)** launched with remote debugging enabled and a dedicated user-data-dir. See [`docs/canary-setup.md`](docs/canary-setup.md).
- **`chrome-devtools` MCP server** configured in your agent host. See the [chrome-devtools-mcp README](https://github.com/ChromeDevTools/chrome-devtools-mcp).
- **Claude Code** or **OpenCode** as the host agent.

## Install

- Claude Code: [`install/claude-code.md`](install/claude-code.md)
- OpenCode: [`install/opencode.md`](install/opencode.md)

Both are a two-file copy.

## Quickstart

After installing, ask your agent:

> "Using chrome-control, read back the title of my currently active tab."

The orchestrator should delegate one call to `chrome-subagent`, which connects to your running browser, reads `document.title`, and returns a single line.

## Repo layout

```
skills/chrome-control/SKILL.md     Orchestrator skill (what to delegate, how to phrase prompts)
agents/chrome-subagent.md          Sub-agent definition (the only component that touches chrome-devtools tools)
docs/architecture.md               Why the split, context-hygiene rationale
docs/safety-model.md               LOW / MED / HIGH tiers, two-gate model
docs/canary-setup.md               Launching Chrome with CDP, profile and port pitfalls
install/claude-code.md             Install paths for Claude Code
install/opencode.md                Install paths for OpenCode
```

## Safety model in one paragraph

Before every action, the sub-agent classifies the action as **LOW** (reversible, audience = self), **MED** (reversible-with-effort or small known group), or **HIGH** (irreversible, multi-user live, or pixel-dependent commit). Every action passes Gate 1 — a DOM read confirming the agent is about to act on the right element. HIGH actions also pass Gate 2 — an inline screenshot confirming surrounding state. When in doubt, escalate one tier. Full details in [`docs/safety-model.md`](docs/safety-model.md).

## License

MIT — see [`LICENSE`](LICENSE).
