# story-creator ↔ task-creator Orchestration Protocol

This document specifies how story-creator invokes task-creator, the expected
inputs and outputs at each boundary, and the rules for dependency-aware task
chain generation.

---

## Overview

```
User / PRD
    │
    ▼
story-creator (Claude Code)
    │  writes
    ▼
.ai/stories/STORY-NNN/
  story.md      ← canonical story definition
  context.md    ← initialized (empty template)
  tasks/        ← empty directory
  logs/         ← empty directory
    │
    ▼
task-creator (Claude Code) ← invoked by story-creator
    │  reads story.md + context.md
    │  writes
    ▼
.ai/stories/STORY-NNN/tasks/
  TASK-001.md
  TASK-002.md
  ...
  TASK-NNN.md
    │
    ▼
Cline (executor)
    │  reads TASK-NNN.md
    │  implements
    │  appends to context.md
    ▼
.ai/stories/STORY-NNN/context.md  ← append-only log
```

---

## Invocation Contract

### story-creator → task-creator

story-creator MUST complete the following before invoking task-creator:

**Preconditions (verified before invocation):**
1. `story.md` is written and all 9 sections are populated.
2. `context.md` is initialized from `templates/context.md`.
3. `tasks/` directory exists (may be empty).
4. `logs/` directory exists.
5. All Anti-Pattern checks pass (SANTI-001 through SANTI-010).
6. Story sizing gates pass (all 5 gates from `docs/sizing-guide.md`).

**Invocation:**
```
/task-creator STORY-NNN
```

task-creator reads:
- `.ai/stories/STORY-NNN/story.md` — for decomposition input
- `.ai/stories/STORY-NNN/context.md` — to avoid re-creating completed work

task-creator writes:
- `.ai/stories/STORY-NNN/tasks/TASK-NNN.md` for each task

**Post-conditions (verified after invocation):**
1. All task files exist in `tasks/`.
2. Task IDs are sequential (TASK-001, TASK-002, …).
3. Dependency chain is acyclic.
4. No task references files outside its Allowed Files list.
5. All tasks include a Context Update section.

---

## Input Specification — story.md to task-creator

task-creator reads `story.md` and uses these sections as input:

| story.md Section | task-creator Usage |
|---|---|
| Story Goal | Becomes the overarching objective constraint |
| Constraints | Applied verbatim to every task's Forbidden section |
| Affected Areas | Seed list for Allowed Files across all tasks |
| Technical Decisions | Prescribes exact implementations (class names, routes, types) |
| Acceptance Criteria | Story-level criteria that individual tasks must contribute to |
| Execution Strategy | Task chain blueprint; defines entry point and dependency order |

---

## Output Specification — task-creator to story-creator

After task-creator completes, story-creator verifies:

```
TASK-NNN.md contains:
  - Objective: specific, no deferred decisions
  - Allowed Files: exact paths (≤ 5 per task)
  - Forbidden: explicit prohibitions + "All files not listed in Allowed Files"
  - Requirements: ≥ 3 MUST/MUST NOT items
  - Acceptance Criteria: ≥ 2 runnable checks including the build command
  - Dependencies: explicit TASK-IDs or "None"
  - Validation Steps: runnable shell commands
  - Context Update: verbatim append block for context.md
```

---

## Dependency Generation Logic

story-creator passes the Execution Strategy to task-creator as the canonical
dependency blueprint. task-creator enforces this logic when assigning IDs:

### Canonical dependency order

```
[No deps]  Data model task(s)         → lowest IDs (001, 002, ...)
           Migration task              → after data model
           Types task                  → after data model
           Repository task             → after types
           Service task                → after repository
           Validator task              → after types (parallel with repository)
           API route tasks             → after service
           Hook tasks                  → after types + API routes
           UI component tasks          → after hooks + types
           Page tasks                  → after components
           Test tasks                  → highest IDs
```

### Cross-story dependencies

When a story depends on another story (listed in story.md Dependencies):
- task-creator treats the upstream story's final output artifacts as pre-existing.
- task-creator does NOT re-create artifacts from upstream stories.
- context.md records which story produced which artifact — task-creator reads this.

Example: STORY-002 (API) depends on STORY-001 (Data Layer).
task-creator for STORY-002 reads context.md and finds:
```
## TASK-003 — <Entity> Repository (completed YYYY-MM-DD)
### Files Changed
- `src/lib/<entity>-repository.ts` — created
```
task-creator for STORY-002 then lists `src/lib/<entity>-repository.ts` in the
Dependencies section of the API route task — it does NOT recreate the repository.

---

## Context Update Protocol

This protocol governs how Cline (the executor) updates `context.md` after
completing each task. story-creator initializes the file; every task's
Context Update section enforces the append schema.

### Append Schema

```markdown
## TASK-NNN — <task name> (completed YYYY-MM-DD)

### Decisions
- <decision made, deviation from plan, or "No deviations">

### Files Changed
- `exact/path/to/file.ts` — created | modified | deleted

### Notes
- <anything the next task executor must know before starting>
```

### Enforcement

- The Context Update section in each task file contains this schema VERBATIM.
- Cline copies this block into context.md after task completion.
- Cline MUST NOT append anything outside this schema to context.md.
- story-creator verifies the schema is present in all tasks after task-creator runs.

### Why append-only

- Overwrites destroy execution history — blocked.
- Edits to previous entries cause merge conflicts — blocked.
- Raw diffs or inline code bloat the context — blocked.
- Only the schema's three sections (Decisions, Files Changed, Notes) are allowed.

---

## Multi-Story Orchestration

When story-creator generates multiple stories for a single feature:

### Sequencing rule

Run stories in dependency order. Never run a UI story before its API story.

```
STORY-001  <Entity> Data Layer Foundation    → run first (no deps)
STORY-002  <Entity> API Layer                → run after STORY-001 complete
STORY-003  <Entity> List View                → run after STORY-002 complete
```

### Cross-story context reads

Before Cline starts any task in STORY-002, it reads:
- `.ai/stories/STORY-001/context.md` — to find produced artifacts.
- `.ai/stories/STORY-002/context.md` — for its own story's state.

task-creator (when generating STORY-002 tasks) reads STORY-001's context.md
to resolve `[TBD: STORY-001]` placeholders into actual file paths.

### Global architecture decisions

Architecture decisions that span multiple stories go in:
```
.ai/architecture.md § Architecture Decisions
```

story-creator reads this section during Phase 0 and must not contradict it.
If a new story requires a decision that conflicts with a recorded decision,
story-creator flags the conflict and asks the user to resolve it before writing.

---

## Error Conditions

| Condition | story-creator behavior |
|---|---|
| story.md has empty Technical Decisions | Stop. Report SANTI-005 violation. |
| story.md story goal is vague | Stop. Report SANTI-002 violation. |
| task-creator produces > 7 tasks | Stop. Report SANTI-001. Re-split the story. |
| task has circular dependency | Stop. Report cycle. Ask user to resolve. |
| context.md is missing | Re-initialize from template. Log warning. |
| upstream story context.md is missing | Stop. Ask user to run upstream story first. |
| task references file from another story's Allowed Files | Flag conflict. Resolve before writing. |
