---
name: quick-task
description: >
  Ultra-lightweight execution mode for small, fast, low-risk tasks. Claude
  executes the change directly — no story, no task files, no executor handoff,
  no orchestration. Triggers ONLY on the explicit /quick-task command; it is
  the single sanctioned path for Claude to write production code directly
  (see CLAUDE.md § Direct Execution Exception). Use for a tiny refactor, small
  file edit, quick fix, one-shot code generation, config tweak, or small doc
  update that does not warrant the full Architect → Story → Task → Executor
  workflow.
version: 1.0.0
---

# quick-task — Direct Execution Mode

A bypass lane for work too small to justify the mono-workflows orchestration
pipeline. When this skill is active, Claude stops being a workflow orchestrator
and behaves like a senior engineer making one focused, contained edit.

**This skill executes. It does not plan, decompose, or delegate.**

---

## Role Override (scoped)

The project's default role (see `CLAUDE.md`) is **architect-planner** — Claude
plans, Cline executes. `quick-task` deliberately overrides that role **for the
duration of one small task only**.

- Inside a passing size gate → Claude edits production code directly.
- The override ends when the task ends. The next request re-enters the default
  architect-planner role unless `/quick-task` is invoked again.
- If the size gate fails at any point → the override is void; hand back to the
  normal workflow (see Escalation Rules).

Invoking `/quick-task` is the user's explicit authorization for this override.

---

## When to Use

Use `quick-task` for:

- tiny refactors (rename, extract a constant, inline a variable)
- small file edits (≤ 2 files)
- quick bug fixes with an obvious cause and fix
- simple, self-contained code generation (one function, one small component)
- lightweight analysis ("what does this function do?", "where is X used?")
- tiny documentation updates (fix a typo, update a code block, add a note)
- configuration adjustments (flip a flag, bump a value, add an entry)
- one-shot tasks completable in a single response

Do NOT use `quick-task` for:

- large features or anything user-facing and multi-part
- architecture design or any new architectural decision
- multi-step implementation that needs progress tracking
- risky refactors (shared modules, public contracts, type-wide changes)
- cross-module / cross-domain coordination
- long-running tasks that span multiple responses
- complex debugging (unknown root cause, requires investigation chains)
- large-context operations (must read many files to be safe)

If a request is on the "do NOT" list → escalate immediately (see below).

---

## Activation Conditions

The skill activates **only** when the user explicitly runs
`/quick-task <description>`.

Per `CLAUDE.md § Direct Execution Exception`, this is the **single** path by
which Claude may execute production code directly. Claude MUST NOT enter
quick-task mode by inferring that a plain request "looks small" — without an
explicit `/quick-task` invocation, the default architect-planner role applies
and the request is planned, not executed.

Once invoked, the task must still pass the Task Size Gate below. If it fails,
the skill escalates to `story-creator` (see Escalation Rules).

---

## Task Size Gate

Run this gate **before touching anything**. The task stays in quick-task mode
only if it passes **every** check. One failure → escalate.

| # | Heuristic | Pass condition |
|---|-----------|----------------|
| 1 | **File count** | ≤ 2 files created or modified |
| 2 | **LOC touched** | ≤ ~40 net lines changed |
| 3 | **Module span** | 1 module / domain only |
| 4 | **Layer span** | 1 layer (does not cross data → service → API → UI) |
| 5 | **Architectural impact** | None — no new pattern, no decision to make |
| 6 | **Dependency surface** | No new dependency; no public contract / shared type change |
| 7 | **Risk class** | Not auth, migrations, infra, CI, payments, or security-sensitive |
| 8 | **Execution time** | Completable in a single response, no progress tracking needed |
| 9 | **Certainty** | The change and its location are known up front — no investigation chain |
| 10 | **Reversibility** | Git-tracked and trivially revertable |

Rule of thumb: **if you have to think about how to decompose it, it is not a
quick task.** When a check is ambiguous, treat it as a failure and escalate —
under-planning a real feature is more expensive than over-routing a small one.

---

## Workflow

Keep every phase tight. The skill's value is speed and low token cost.

### Phase 0 — Triage

Run the Task Size Gate. **Pass** → continue to Phase 1. **Fail** → Escalation Rules.
Do not announce the gate result unless escalating; just proceed.

### Phase 1 — Locate

Read only the file(s) the task touches. Do not survey the codebase. If you
cannot identify the target file(s) without broad exploration → fail check 9,
escalate.

### Phase 2 — Execute

Make the change directly with `Edit` / `Write`. One focused edit. No
`.ai/` artifacts, no story/task/context files, no TodoWrite for a single edit.

### Phase 3 — Verify

Do the cheapest sufficient check:

- code change → run lint/build/test for the touched scope if a command exists
  and is fast; otherwise re-read the edited region.
- doc/config change → re-read the edited region.

If verification reveals the change is bigger than expected → escalate.

### Phase 4 — Report

One to three lines: what changed, where (`file:line`), and the verify result.
No summary headers, no restating the diff. Stop.

---

## Execution Rules

When `quick-task` is active, Claude MUST:

- execute the task directly — be the engineer, not the orchestrator
- NOT invoke `story-creator`, `task-creator`, `tier-classifier`, or executor orchestration
- NOT create stories, tasks, context files, or any `.ai/` artifact
- NOT delegate to Cline or spawn sub-agents
- NOT produce planning markdown or decomposition documents
- minimize verbosity — no preamble, no trailing summary beyond Phase 4
- minimize token use — read narrowly, edit surgically
- optimize for speed and a single-response turnaround

Logging is **off by default** (zero artifacts). If the user explicitly wants a
trace, append one line to `.ai/quick-tasks/quick-log.md`:
`### [<ISO date> <HH:MM>] quick-task — <what changed> — <file(s)>`.

---

## Safety Constraints

- Never use `quick-task` to skip planning that the work genuinely needs. The
  gate is a filter, not a shortcut to dodge orchestration.
- Never silently expand scope. If the task grows, stop and escalate — do not
  "just finish it" past the gate.
- Respect git safety: no destructive git commands, no force operations, no
  committing unless the user asks.
- For anything risky, irreversible, or shared-state-affecting → escalate or ask
  first, even if it is technically small.
- Stay inside the located file(s). Touching a third file is a gate failure.
- If verification fails and the fix is not obvious within scope → escalate;
  do not start a debugging chain inside quick-task mode.

---

## Escalation Rules

Escalate back to the normal mono-workflows pipeline the moment **any** of these
becomes true — before, or during, execution:

- file count, LOC, module span, or layer span exceeds the size gate
- multiple domains / modules turn out to be affected
- an architectural decision is required
- the change touches a public contract, shared type, or new dependency
- context grows large (you need to read many files to proceed safely)
- the outcome becomes uncertain (root cause unknown, fix not obvious)
- the task needs progress tracking across multiple responses
- the task would benefit from parallelization or a verification chain
- risk class turns out to be auth / migration / infra / CI / security

**Escalation procedure:**

1. Stop quick-task mode immediately. Do not partially implement.
2. If a partial edit was already made and is unsafe to leave, revert it or tell
   the user exactly what state the files are in.
3. State in one line why the task exceeded quick-task scope.
4. Hand off to the normal workflow:
   `/story-creator "<original request>"`
   (story-creator Phase 0.5 will classify it into medium / large / epic).
5. Let the user confirm before the heavier workflow runs.

Escalation is a success, not a failure — it means the router worked.

---

## Examples (good fits)

```
/quick-task Fix the typo "recieve" → "receive" in src/components/Inbox.tsx
→ 1 file, 1 word. Phase 0 pass → edit → re-read → "Fixed typo at Inbox.tsx:42."

/quick-task Rename the local var `tmp` to `pendingCount` in calcTotals()
→ 1 file, ~4 lines, no contract change. Direct edit, lint, done.

/quick-task Bump the retry limit from 3 to 5 in config/http.ts
→ Config tweak, 1 line. Direct edit, done.

"quick fix — the empty-state text should say 'No notes yet'"
→ Signal-activated, 1 file, copy change. Gate passes → direct edit.

/quick-task Add a JSDoc comment to the exported formatDate() helper
→ 1 file, doc-only. Direct edit, done.

"what does the debounce() in src/lib/utils.ts actually do?"
→ Lightweight analysis. Read one file, answer. No artifacts.
```

## Anti-Patterns (escalate instead)

```
/quick-task Add user authentication
→ ANTI: risk class = auth, multi-file, architectural. Escalate → story-creator.

/quick-task Refactor the API layer to use the repository pattern
→ ANTI: "refactor (system|architecture)", cross-file, new pattern. Escalate.

/quick-task Add a search feature to the notes list
→ ANTI: feature, UI + backend, multi-layer. Escalate → large/epic tier.

/quick-task Fix the bug where the app crashes sometimes on save
→ ANTI: unknown root cause, investigation chain needed (check 9 fails). Escalate.

/quick-task Just quickly migrate the DB schema, it's a small change
→ ANTI: "migration" risk class, irreversible. Escalate regardless of size.

/quick-task Rename the User type everywhere
→ ANTI: shared type / public contract, unbounded file count. Escalate.
```

**The recurring trap:** a request *described* as small ("just", "quick",
"simple") that *is not* small. Trust the size gate, not the adjective.

---

## Relationship to Planning Tiers

`quick-task` and the `trivial` planning tier overlap but are not the same:

| | `trivial` tier (story-creator) | `quick-task` skill |
|---|---|---|
| Produces | a `QUICK-NNN-<slug>.md` file | nothing — direct edit |
| Executed by | Cline executor | Claude, in-line |
| Use when | you want a tracked, file-scoped task for the executor | you want the change made now, no paperwork |

Pick `quick-task` for immediacy; pick `/story-creator --tier=trivial` when you
still want an executor-handoff artifact. Both escalate the same way when a task
turns out to be bigger than it looked.

---

## Reference Files

- [CLAUDE.md](../../../CLAUDE.md) — default architect-planner role this skill scopes-overrides
- [.ai/planning-tiers.md](../../../.ai/planning-tiers.md) — tier config; `quick-task` ≈ a no-artifact `trivial`
- [story-creator/SKILL.md](../story-creator/SKILL.md) — the workflow to escalate into
- [tier-classifier/SKILL.md](../tier-classifier/SKILL.md) — classifier story-creator runs on escalated tasks
