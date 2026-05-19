# Safety model

Every browser action the sub-agent takes passes through this model.

## Two questions, three tiers

Before any ACT (`click`, `type_text`, `fill`, `press_key`), the sub-agent answers two questions:

1. **Reversibility.** If this turns out to be wrong, can I undo it? Ctrl-Z, version history, message edit, draft autosave → yes. Send, Submit, Delete, Pay, external API write → no.
2. **Audience.** Am I the only person affected, or is this visible to / shared with others right now?

The answers map to a tier:

| Tier  | Conditions                                                       | Required gates              |
|-------|------------------------------------------------------------------|-----------------------------|
| LOW   | Fully reversible AND audience = self                             | Gate 1                      |
| MED   | Reversible-with-effort OR audience = small/known group           | Gate 1 + payload DOM read   |
| HIGH  | Irreversible OR multi-user live OR pixel-dependent commit        | Gate 1 + Gate 2             |

## The gates

### Gate 1 — State check (every ACT, no exceptions)

```
evaluate_script confirms the expected DOM anchor:
  active element, selection target, modal presence,
  URL/title, whatever uniquely identifies
  "I am about to act on the right thing."
~30 tokens. If actual ≠ expected → STOP, return error to parent.
```

The point of Gate 1 is to never act blind. A click without a preceding state check is a guess. The check costs ~30 tokens and catches every "wrong tab", "wrong cell selected", "modal opened over my target" failure cheaply.

### Gate 2 — Visual safety (HIGH only)

```
take_screenshot inline (no filePath).
Confirm visually that the surrounding state
  (co-editors, dialogs, layout)
is what the action assumes.
~1500 tokens. Lives only in the sub-agent's context.
```

Gate 2 catches what Gate 1 cannot: a co-editor's cursor sitting on the same row, a confirmation dialog the DOM read didn't anticipate, a layout shift that moved the target. Reserved for HIGH because at ~1500 tokens it isn't free.

The screenshot lives only in the sub-agent's context and is discarded on return. It does **not** come back to the parent transcript.

## Escalation rule

> **When in doubt, escalate one tier.**

A wrong HIGH classification costs ~1500 tokens. A wrong LOW classification costs corrupted shared data or an un-recallable submission. The asymmetry is enormous; default toward over-classifying.

## Uncertainty at HIGH

If the Gate 2 screenshot does not resolve the doubt — ambiguous state, unexpected dialog, co-editor activity in the target region — the sub-agent does **not** proceed. It stops, returns the relevant reading and the question to the parent, and lets the parent escalate to the user.

This is the only point in the whole pattern where a human is asked to look at the screen. By design, the sub-agent never asks the user a question; it returns to the parent, and the parent decides whether to ask.

## What the parent contributes

The orchestrator has context the sub-agent lacks: who else is using this surface, whether the action is part of a "draft" or "send" flow, whether the user has approved a batch of changes already. That context belongs in the prompt to the sub-agent so its tier classification is informed:

- "Personal scratch document, no co-editors" → invites LOW.
- "Document shared with a team — assume co-editors" → invites HIGH.
- "Form submission, will Submit at the end" → HIGH by definition.

The parent describes audience and reversibility; the sub-agent classifies.

## Failure modes the gates catch

| Failure                                            | Caught by                               |
|----------------------------------------------------|-----------------------------------------|
| Wrong tab in focus                                 | Gate 1 (URL/title check)                |
| Wrong cell / row / element selected                | Gate 1 (active-element check)           |
| Unexpected modal in front of target                | Gate 1 (modal presence boolean)         |
| Stale selector after SPA navigation                | Gate 1 (element count check)            |
| Co-editor cursor on the same target                | Gate 2 (visual)                         |
| Layout shift moved the target visually             | Gate 2 (visual)                         |
| Confirmation dialog after clicking Send            | Gate 2 (visual)                         |
| Selector ambiguous between two similar elements    | Gate 1 (count check) + a unique anchor  |
