# Install for Claude Code

Two files, two destinations.

## 1. Install the chrome-devtools MCP server

Follow the instructions in the [`chrome-devtools-mcp`](https://github.com/ChromeDevTools/chrome-devtools-mcp) README. After install, `claude mcp list` should include it. The MCP exposes tools named `mcp__chrome-devtools__*` inside Claude Code.

## 2. Drop in the skill and the agent

```bash
mkdir -p ~/.claude/skills/chrome-control
mkdir -p ~/.claude/agents

cp skills/chrome-control/SKILL.md ~/.claude/skills/chrome-control/SKILL.md
cp agents/chrome-subagent.md      ~/.claude/agents/chrome-subagent.md
```

The skill is auto-loaded based on its `description:` frontmatter when Claude Code detects a matching task. The agent is invokable via the `Agent` tool whenever the parent decides to delegate.

## 3. Launch Chrome on CDP 9222

See [`docs/canary-setup.md`](../docs/canary-setup.md). The skill and the sub-agent both assume the browser is already up and connectable.

## 4. Verify

In any Claude Code session:

> "Using chrome-control, read back the title of my currently active tab."

Expected behavior:

1. The parent loads the chrome-control skill (no chrome-devtools tools exposed at this layer).
2. The parent dispatches one `Agent` call to `chrome-subagent` with a 4-line delegation prompt.
3. The sub-agent connects to CDP 9222, runs `evaluate_script` returning `document.title`, and returns one line.
4. The parent reports the title to you.

If the parent tries to call chrome-devtools tools directly, the skill text isn't being loaded — check that the skill file lives at the path above and that its frontmatter `description:` field matches the task.

## Updating

Re-run the `cp` commands. Claude Code re-reads the skill/agent files at session start; no daemon to restart.

## Uninstall

```bash
rm -rf ~/.claude/skills/chrome-control
rm    ~/.claude/agents/chrome-subagent.md
```
