# Install for OpenCode

Two files, two destinations.

## 1. Install the chrome-devtools MCP server

Follow the instructions in the [`chrome-devtools-mcp`](https://github.com/ChromeDevTools/chrome-devtools-mcp) README and add it to your OpenCode MCP config. The tools are exposed as `chrome-devtools_*` (note: no `mcp__` prefix in OpenCode).

## 2. Drop in the skill and the agent

```bash
mkdir -p ~/.config/opencode/skills/chrome-control
mkdir -p ~/.config/opencode/agents

cp skills/chrome-control/SKILL.md ~/.config/opencode/skills/chrome-control/SKILL.md
cp agents/chrome-subagent.md      ~/.config/opencode/agents/chrome-subagent.md
```

> **Note on tool names.** The bundled `agents/chrome-subagent.md` lists tools under their Claude Code names (`mcp__chrome-devtools__*`). OpenCode resolves the same MCP tools under the `chrome-devtools_*` prefix. If your OpenCode build requires the `tools:` frontmatter to match the OpenCode-side names, swap the prefix:
>
> ```bash
> sed -i '' 's/mcp__chrome-devtools__/chrome-devtools_/g' \
>   ~/.config/opencode/agents/chrome-subagent.md
> ```
>
> (Drop the `''` after `-i` on Linux.)

## 3. Launch Chrome on CDP 9222

See [`docs/canary-setup.md`](../docs/canary-setup.md). The skill and the sub-agent both assume the browser is already up and connectable.

## 4. Verify

In any OpenCode session:

> "Using chrome-control, read back the title of my currently active tab."

Expected behavior:

1. The parent loads the chrome-control skill (no chrome-devtools tools exposed at this layer).
2. The parent dispatches one `Task` call with `subagent_type: chrome-subagent` and a 4-line delegation prompt.
3. The sub-agent connects to CDP 9222, runs `evaluate_script` returning `document.title`, and returns one line.
4. The parent reports the title to you.

If the parent tries to call `chrome-devtools_*` tools directly, the skill text isn't being loaded — check the path above and the frontmatter `description:`.

## Updating

Re-run the `cp` commands. OpenCode re-reads the skill/agent files at session start.

## Uninstall

```bash
rm -rf ~/.config/opencode/skills/chrome-control
rm    ~/.config/opencode/agents/chrome-subagent.md
```
