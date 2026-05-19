# Architecture

## Two components, one job

```
┌──────────────────────────┐         ┌────────────────────────────┐
│  Parent agent            │         │  chrome-subagent           │
│  (orchestrator)          │         │  (executor)                │
│                          │         │                            │
│  Loads skill:            │  task   │  Owns chrome-devtools MCP  │
│   chrome-control         │ ──────▶ │  Connects to CDP 9222      │
│                          │         │  READ / ACT / CAPTURE      │
│  Decomposes work         │ ◀────── │  Returns ≤3 lines          │
│  Stitches outcomes       │ outcome │                            │
└──────────────────────────┘         └────────────────────────────┘
                                              │
                                              │ CDP
                                              ▼
                                     ┌─────────────────────┐
                                     │  User's Chrome      │
                                     │  --remote-debugging │
                                     │  -port=9222         │
                                     └─────────────────────┘
```

The orchestrator never touches the chrome-devtools tools directly. The sub-agent is the only component holding them. This is enforced by two mechanisms:

1. **Skill text.** `skills/chrome-control/SKILL.md` explicitly tells the parent not to call `chrome-devtools_*` itself.
2. **Sub-agent tool list.** `agents/chrome-subagent.md` declares the chrome-devtools tools in its `tools:` frontmatter, so the host grants them only inside the sub-agent's context. The parent's tool inventory does not include them.

## Why this split exists

### Context hygiene

A single `take_snapshot` of a complex SaaS page is roughly 8000 tokens of accessibility-tree text. A typical multi-step browser task involves several snapshots, several screenshots, and dozens of `evaluate_script` reads. If all of that lives in the parent's context, three things go wrong:

- The parent's working memory fills up with stale DOM that has nothing to do with the original task.
- Subsequent reasoning quality drops as the model spends attention on irrelevant tokens.
- Cost grows linearly with browser activity.

Sub-agents in both Claude Code and OpenCode discard their context on return. Whatever the executor read, snapshotted, or screenshotted is gone the moment it returns its outcome line. The parent gets a clean three-line summary and moves on.

### Safety boundary

A second, equally important benefit: the safety logic lives in one place. The sub-agent prompt forces every action through Gate 1 (DOM read confirming the right target) and HIGH actions through Gate 2 (inline screenshot of surrounding state). The orchestrator can't accidentally bypass these gates, because it can't reach the tools at all. Every browser action in the system goes through the same gauntlet.

## Decomposition contract

The orchestrator's job is to break a user request into single-purpose sub-tasks and dispatch them one at a time. The 4-line delegation template in `skills/chrome-control/SKILL.md` is the canonical form:

```
Page: <url or "the open tab on app X">
Action: <one concrete operation>
Verify: <one DOM value to read back>
Return: <one-line outcome, ≤3 lines total>
```

Each sub-task is self-contained — the executor cannot ask clarifying questions. If the prompt is ambiguous, the executor stops and returns the ambiguity. The orchestrator then refines and retries.

## Serialization

Sub-agents have no view of each other. If two run in parallel and the browser isn't already up, each tries to launch it independently, producing visible window flicker and risking a clobbered profile. The orchestrator must serialize chrome-subagent calls — never spawn two in parallel against the same Chrome instance.

## Direct-drive escape hatch

For interactive debugging where you need to watch each step happen, you can disable the split and call the chrome-devtools tools from the parent directly. This forfeits context hygiene; use it sparingly and only when iterating on a new selector or recipe.
