---
name: intent-verification
description: >
  Pre-orchestration behavior gate. Before Claude creates any story, task,
  epic, workflow, or Cline delegation, this gate confirms what the user
  actually wants — a direct answer, a quick direct fix, a planning/architecture
  discussion, or full workflow orchestration. Run it at the start of any
  request that is NOT an explicit command, especially short or vague ones
  ("fix this", "optimize this", "add this feature", "make it better",
  "refactor this", "support dynamic workflows"). Invocable via /intent-verify.
version: 1.0.0
---

# intent-verification — Pre-Orchestration Intent Gate

Stops premature orchestration. A vague request must not silently become a
story, a task chain, or a Cline delegation. This gate decides **which mode** a
request belongs to, asking the user only when the request is genuinely
ambiguous.

**This skill routes. It does not plan, classify tiers, or implement.**

It reads `.ai/intent-verification.md` for all signals, modes, and routing
policy — that file is the single source of truth.

---

## Position in the Pipeline

```
request ─▶ [intent-verification] ─▶ Direct Answer | Mode 1 | Mode 2 | Mode 3
```

- Runs **before** `tier-classifier`. Intent decides the *mode*; tier-classifier
  decides *depth* inside Mode 3 only.
- Runs **before** `story-creator`, `task-creator`, and `/quick-task`.
- It is the outermost gate of the Architect → Workflow → Executor pipeline.

---

## Invocation

```
/intent-verify "optimize this workflow"
```

Also runs automatically — mandated by `CLAUDE.md § Intent Verification` — at
the start of any request that is not an explicit command.

---

## Workflow

Keep every phase fast. Most requests resolve in Phase 1 or 2 with **no question
and near-zero tokens**. Only Phase 3 talks to the user.

### Phase 0 — Load

Read `.ai/intent-verification.md`. If missing, stop and tell the user to create
it — do not guess routing.

### Phase 1 — Fast-Path Check

If the request matches a row in `§ Clear-Intent Fast Paths` (explicit command
or explicit mode phrasing) → route there **immediately, no question**. Done.

### Phase 2 — Ambiguity Detection

Evaluate `§ Ambiguity Signals`. If **no** signal fires → intent is clear; route
silently to the matching mode and proceed. Done.

### Phase 3 — Clarify (only if ambiguous)

Ask **one** concise question with `AskUserQuestion`. Offer only the modes that
plausibly fit (often two, not all three). Short labels, no lecturing:

- **Quick fix** — I edit it directly now, no tracking
- **Plan first** — we design / discuss before any code
- **Full workflow** — generate stories + tasks for Cline

One question. Then route on the answer.

### Phase 4 — Route

| Answer / outcome | Hand off to |
|---|---|
| Quick fix | `/quick-task <request>` (Mode 1) |
| Plan first | Mode 2 — discuss in conversation, no files |
| Full workflow | `/story-creator <request>` → Phase 0.5 tier-classifier (Mode 3) |
| It was a question | Answer it directly (Direct Answer) |

If the user does not answer, or is still unclear → default to the **lightest**
plausible outcome. Never default into orchestration.

---

## Trigger Heuristics

Verify intent when **any** of these holds (full list in
`.ai/intent-verification.md § Ambiguity Signals`):

- request is short with no object/scope — "fix this", "make it better"
- open-ended verb, no size — "improve", "optimize", "refactor", "support X"
- deliverable unnamed — code? plan? answer? unclear
- size unknown without investigation
- no explicit command and no explicit mode phrasing
- request plausibly fits two or more modes

Skip verification (route silently) when none hold, or a fast path matches.

---

## Escalation / Mode-Switch Rules

See `.ai/intent-verification.md § Mode-Switch Rules`. In short:

- Mode 1 `/quick-task` size gate fails → switch to Mode 3.
- Mode 2 discussion ends with "now build it" → route to Mode 3 (re-verify only
  if scope is still unclear).
- Mode 3 tier-classifier returns `trivial` and the user wants it now → offer
  Mode 1 instead.
- Request turns out to be a question → Direct Answer.

A mode switch mid-flight is expected and healthy — the gate self-corrects.

---

## Example Conversations

**Ambiguous → clarify, then route light**

```
User: "optimize this workflow"
Gate: signals fire — open-ended verb, no scope, no mode.
Ask:  "Want me to (a) tweak it directly, (b) discuss what 'optimized' should
       mean first, or (c) plan it as a tracked workflow?"
User: "just discuss it"
→ Mode 2. No files created.
```

**Clear command → silent route, no question**

```
User: "/quick-task rename `tmp` to `count` in utils.ts"
Gate: fast path — explicit command. → Mode 1. No question.
```

**Question phrasing → Direct Answer**

```
User: "what does the debounce helper do?"
Gate: fast path — question phrasing. → Direct Answer. No question.
```

**Vague feature → clarify, do NOT auto-orchestrate**

```
User: "add a search feature"
Gate: "feature" + no detail = ambiguous.
Ask:  "Plan this as a full workflow (stories + tasks for Cline), or do you
       want a lightweight version I build directly?"
User: "full workflow"
→ Mode 3 → /story-creator → tier-classifier (likely `large`).
```

---

## Positive Examples (gate working)

```
"fix this"            → ambiguous → ask → route to the answer
"/story-creator …"    → fast path → Mode 3, no question
"explain this regex"  → fast path → Direct Answer, no question
"refactor this file"  → ambiguous (scope unknown) → ask → route
"plan the auth flow"  → fast path ("plan") → Mode 2, no question
```

## Anti-Pattern Examples (what the gate prevents)

```
ANTI — orchestrating a vague prompt
  User: "refactor this"
  WRONG: immediately run /story-creator, generate STORY-005 + 6 tasks.
  RIGHT: ask one question — scope and mode are unknown.

ANTI — asking when intent is explicit
  User: "/story-creator add CSV export"
  WRONG: "Do you want a workflow or a quick fix?" — they already said.
  RIGHT: route straight to Mode 3.

ANTI — defaulting upward
  User: "make this better"  (no answer to the clarification)
  WRONG: assume Mode 3, build stories.
  RIGHT: default to the lightest plausible outcome (Direct Answer / Mode 1).

ANTI — multi-round interrogation
  WRONG: ask 4 questions about requirements before routing.
  RIGHT: one question, route, gather detail inside the chosen mode.

ANTI — assuming Cline
  User: "add this feature"
  WRONG: assume full workflow + Cline delegation.
  RIGHT: ask — the user may want a lightweight direct build.
```

---

## Constraints

- This skill MUST NOT create or modify files — it only routes.
- It MUST NOT create stories, tasks, epics, or quick-task files.
- It MUST NOT ask when a fast path matches or no ambiguity signal fires.
- It MUST NOT exceed one clarification round before defaulting light.
- It MUST NOT assume Cline execution or orchestration by default.
- It MUST NOT invoke `tier-classifier` — tiers are decided inside Mode 3 only.
- If `.ai/intent-verification.md` is missing, stop and report.

---

## Reference Files

- [.ai/intent-verification.md](../../../.ai/intent-verification.md) — signals, modes, routing (single source of truth)
- [CLAUDE.md](../../../CLAUDE.md) — mandates this gate before any planning
- [quick-task/SKILL.md](../quick-task/SKILL.md) — Mode 1 target
- [story-creator/SKILL.md](../story-creator/SKILL.md) — Mode 3 target
- [tier-classifier/SKILL.md](../tier-classifier/SKILL.md) — runs after, inside Mode 3
