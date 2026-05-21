# Intent Verification

Behavior layer that runs **before any orchestration**. It stops Claude from
creating stories, tasks, epics, or workflows — or delegating to Cline — while
the user's request is still ambiguous, underspecified, or possibly lightweight.

This file is the **single source of truth** for intent-verification behavior:
ambiguity signals, the execution modes, and routing. The `intent-verification`
skill reads this file — do not duplicate heuristics in skill code.

---

## Position in the Pipeline

```
User request
   │
   ▼
┌──────────────────────┐
│ Intent Verification  │  ← decides the MODE
└──────────────────────┘
   │
   ├─ Direct Answer ─▶ just answer the question, no mode, no orchestration
   ├─ Mode 1 ────────▶ /quick-task .............. direct execution, no artifacts
   ├─ Mode 2 ────────▶ planning / architecture ... discussion only, no artifacts
   └─ Mode 3 ────────▶ /story-creator ─▶ Phase 0.5 tier-classifier ─▶ tier path
```

Intent Verification decides **which mode** a request belongs to.
`tier-classifier` decides **how deep** to plan *within Mode 3*. Different
questions, run in that order. Never jump straight to tier classification.

---

## Execution Modes

| Mode | Name | Claude does | Skills / tools | Artifacts |
|---|---|---|---|---|
| 1 | Quick Direct Execution | Makes the change itself, fast | `/quick-task` | none |
| 2 | Planning / Architecture | Discusses, designs, weighs options | conversation; optionally `/tier-classify` for sizing | none unless asked |
| 3 | Full Workflow Orchestration | Generates stories + tasks, delegates to Cline | `/story-creator` → `task-creator` | `.ai/stories\|epics/…` |

**Direct Answer** is not a mode — if the request is simply a question, answer
it. No mode, no routing, no orchestration.

---

## Ambiguity Signals

Run intent verification (ask before routing) when **any** signal is present.

### Clarity
- Request is short and lacks an object or scope — "fix this", "make it better".
- Verb is open-ended with no size or deliverable named — "improve", "optimize",
  "enhance", "refactor", "clean up", "handle X better", "support X".
- The deliverable is unnamed — unclear whether the user wants code, a plan, a
  discussion, or an answer.

### Scope
- Implementation size is unknown without investigation.
- The request could touch one file or twenty — the text does not say.
- "add a feature" / "add X" with no detail on surface area.

### Mode
- No explicit command (`/quick-task`, `/story-creator`, …) was used.
- No explicit mode phrasing — "just do it", "plan only", "only answer",
  "don't write code", "create stories".
- The request plausibly fits **two or more** modes.

If **no** signal fires, intent is clear — route silently, do not ask.

---

## Clear-Intent Fast Paths (do NOT ask)

Skip the clarification question entirely when intent is explicit:

| Signal in the request | Route to |
|---|---|
| `/quick-task …` | Mode 1 |
| `/story-creator …`, `/task-creator …`, "create a story/tasks for …" | Mode 3 |
| `/tier-classify …` | classification only |
| "what is…", "how does…", "explain…", "just answer" | Direct Answer |
| "plan only", "don't implement", "let's discuss the design" | Mode 2 |
| "have Cline build it", "run the full workflow", "make it a tracked task" | Mode 3 |
| Request already names exact file(s) + exact change | Mode 1 (if small) |

When the user has already stated the mode, asking again is friction — don't.

---

## Routing Decision

After signals are evaluated:

1. **No ambiguity** → route silently to the matching mode. Proceed.
2. **Ambiguous** → ask **one** concise clarification question offering only the
   plausible modes (often two, not always all three). Route on the answer.
3. **Still ambiguous after one question** → default to the *lightest* plausible
   outcome (Direct Answer > Mode 1 > Mode 2 > Mode 3). Never default upward.

**Bias rule:** when unsure between two modes, pick the lighter one.
Under-orchestrating costs one follow-up message; over-orchestrating costs
wasted stories, tasks, tokens, and user trust.

---

## Mode-Switch / Escalation Rules

Intent can be misjudged. Re-verify and switch modes when reality diverges:

- **Mode 1 → Mode 3**: the `/quick-task` size gate fails (see quick-task
  Escalation Rules). Stop, hand to `/story-creator`.
- **Mode 2 → Mode 3**: discussion concludes and the user says "now build it".
  Re-enter verification only if scope is still unclear; otherwise route.
- **Mode 3 → Mode 1**: tier-classifier returns `trivial` and the user wants it
  done now rather than tracked → offer `/quick-task` instead of a quick-task file.
- **Any → Direct Answer**: the "request" turns out to be a question.

A mode switch is a success signal — the gate is self-correcting.

---

## What This Layer Must NOT Do

- Must not ask a clarification question when intent is already clear.
- Must not ask more than one round of questions before defaulting light.
- Must not create stories, tasks, epics, or quick-task files before a mode is
  confirmed.
- Must not assume Cline execution, or that orchestration is wanted, by default.
- Must not over-engineer a small request into a workflow.

---

## Extending

- To change ambiguity detection, edit **Ambiguity Signals** here.
- To change what each mode produces, edit **Execution Modes** here.
- The `intent-verification` skill holds only the *procedure* (detect → ask →
  route) and examples. Keep all policy in this file — do not duplicate it.
