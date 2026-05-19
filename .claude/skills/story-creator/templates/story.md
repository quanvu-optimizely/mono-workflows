# STORY-NNN-<slug> — <Entity> <Concern> [Layer]

## Story Goal

<!-- One sentence. Active subject (user or system). Observable outcome, not implementation step. -->

## Business Context

<!-- 2–5 sentences. Why does this story exist? What product requirement drives it?
     No implementation details here. -->

## Constraints

**Stack constraints:**
_(Copy the Stack Rules block verbatim from `.ai/architecture.md § Stack Rules`)_

**Scope constraints:**
- This story owns: <!-- exact files/directories this story controls -->
- This story MUST NOT modify: <!-- exact files/directories that are out of scope -->
- Auth is out of scope (unless this is the Auth Setup story)
- Global state management is out of scope (unless explicitly in story goal)

## Affected Areas

**New files (created by this story's tasks):**
- `<!-- exact/path/to/file.ts -->` — <!-- reason -->

**Modified files:**
- `<!-- exact/path/to/file.ts -->` — <!-- what changes -->

## Technical Decisions

<!-- All architectural decisions the executor MUST follow. No deferred choices.
     Remove sub-headings for layers not relevant to this story. -->

**Data layer:**
- <!-- Exact model name, field names, field types -->
- <!-- Migration strategy: additive-only, no destructive changes -->

**Service layer:**
- <!-- Exact class/function names, method signatures, error types thrown -->

**API layer:**
- <!-- Exact HTTP method, route path, request/response shape (TypeScript interface) -->

**UI layer:**
- <!-- Exact component name, props interface, token/class names used -->

## Acceptance Criteria

- [ ] `<build-command>` exits with code 0 after all tasks complete
- [ ] <!-- Named export or curl command for the primary deliverable -->
- [ ] <!-- Type-check or runtime verification -->

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| <!-- risk description --> | Low / Med / High | <!-- mitigation strategy --> |

## Dependencies

<!-- List each upstream story with the artifact it produces.
     If none: write "None" -->

- STORY-NNN-<slug> — <!-- title --> — produces `<!-- artifact -->` consumed by this story

## Execution Strategy

**Entry task**: TASK-NNN (no dependencies — run first)

**Task chain:**

```
TASK-001  [no deps]       <!-- task name -->
  ↓
TASK-002  [deps: 001]     <!-- task name -->
  ↓
TASK-003  [deps: 002]     <!-- task name -->
```

**Parallelism notes:**
- <!-- Which tasks can run in parallel and why -->

**Context update protocol:**
After each task completes, the executor MUST append the structured block from
that task's "Context Update" section to `.ai/stories/STORY-NNN-<slug>/context.md`
under "Completed Tasks". The next task executor MUST read `context.md` in
full before starting.
