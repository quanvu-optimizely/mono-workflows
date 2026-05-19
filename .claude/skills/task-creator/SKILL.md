---
name: task-creator
description: >
  Transform a feature or story description into a set of highly constrained,
  file-scoped implementation tasks for execution agents (Cline). Triggers on
  /task-creator command or when the user asks to "create tasks for STORY-XXX",
  "generate tasks from story", or "break down this feature into tasks". Produces
  deterministic, bounded tasks with explicit file scope, acceptance criteria,
  and dependency chains.
version: 1.1.0
---

# task-creator — Implementation Task Generator

Transforms a story or feature description into deterministic, file-scoped tasks
for execution by Cline or any other constrained agent.

**You are operating in architect-planner mode. You write task files only. You
MUST NOT write or modify production source code outside `.ai/`.**

---

## Stack Reference

Read `.ai/architecture.md § Stack` and `.ai/architecture.md § Stack Rules`.

Every task involving UI files MUST include the applicable rules from
`.ai/architecture.md § Stack Rules` in the task's `## Requirements` section.

---

## Domain Vocabulary

Read `.ai/architecture.md § Domain`. Use entity names exactly as defined there
in task titles and file paths.

---

## Invocation Patterns

```
/task-creator STORY-001-<slug>
/task-creator STORY-001-<slug> --dry-run
/task-creator "Add search to the entity list"
/task-creator STORY-001-<slug> --split TASK-003
/task-creator STORY-001-<slug> --append
```

| Flag | Behavior |
|---|---|
| `STORY-NNN-<slug>` | Read story from `.ai/stories/STORY-NNN-<slug>/story.md` |
| `"<description>"` | Use inline description; auto-create story directory |
| `--dry-run` | Print tasks to conversation; do NOT write files |
| `--split <TASK-ID>` | Re-split an oversized task into sub-tasks |
| `--append` | Add tasks to existing story without overwriting |

---

## Workflow

### Phase 0 — Locate Input

1. **Read `.ai/architecture.md`** — load stack constraints, domain vocabulary,
   path conventions, and open architecture decisions. Stop if the file is
   missing and tell the user to create it.
2. If given `STORY-NNN-<slug>`, read `.ai/stories/STORY-NNN-<slug>/story.md`.
   - If missing, stop and tell the user to create it first (or use story-creator).
3. If given an inline description:
   - Find the highest `STORY-NNN` under `.ai/stories/` and increment.
   - Generate a kebab-case slug (3–5 words, lowercase, drop articles/prepositions).
   - Create `.ai/stories/STORY-NNN-<slug>/story.md` with the description.
   - Create `.ai/stories/STORY-NNN-<slug>/context.md` (empty template).
4. Read `.ai/stories/STORY-NNN-<slug>/context.md` — it contains prior execution agent
   decisions. These constrain the task set (do not re-create already-completed work).

### Phase 1 — Domain Decomposition

Identify every distinct concern in the story. Each concern becomes exactly one
task (or is split if it fails the size gate). Apply the Concern Taxonomy:

| Concern | Examples | Task? |
|---|---|---|
| Data model | DB schema field/model | 1 task per model |
| DB migration | schema migration command | 1 task (after schema task) |
| Repository | DB access layer | 1 task per entity |
| Service | Business logic | 1 task per service module |
| Validator | Input validation util | 1 task per validator module |
| Types | Shared interfaces/enums | 1 task (may cover 1–3 files) |
| Server action | Server-side action handler | 1 task per action group |
| API route | REST handler | 1 task per HTTP method |
| Hook | Custom framework hook | 1 task per hook |
| UI component | UI component | 1 task per component |
| UI page | Page/layout | 1 task per page |
| Config | Env vars, config files | 1 task |
| Test suite | Unit/integration tests | 1 task per tested module |

### Phase 2 — Dependency Graph

Build an explicit dependency ordering:

```
Data model
  └─ Migration
  └─ Types
       └─ Repository
            └─ Service
                 └─ API route / Server action
                      └─ Hook
                           └─ UI component
                                └─ Page
                                     └─ Tests
```

Rules:

- Data model tasks have NO dependencies (run first, lowest IDs).
- Type tasks depend on data model tasks.
- Repository tasks depend on type tasks.
- Service tasks depend on repository tasks.
- API/action tasks depend on service tasks.
- Hook tasks depend on type tasks and may depend on API tasks.
- Component tasks depend on type tasks and hooks.
- Page tasks depend on component tasks.
- Test tasks have the highest IDs and depend on the module under test.

Assign IDs `TASK-001`, `TASK-002`, … in dependency order. Lower ID = runs sooner.

### Phase 3 — Task Sizing Check

For each proposed task, run ALL four gates. Fail any gate → split the task.

```
FILE COUNT  : ≤ 5 files
DOMAIN COUNT: = 1 concern type
LOC ESTIMATE: ≤ 200 lines changed
UPSTREAM DEPS: ≤ 3 tasks
```

See `docs/sizing-guide.md` for splitting patterns.

### Phase 4 — Task File Generation

Write each task as `.ai/stories/STORY-NNN-<slug>/tasks/TASK-NNN-<slug>.md`.

The task slug is derived from the task title using the same rules as story slugs:
3–5 words, lowercase, hyphen-separated, drop articles and prepositions.
Example: `<Entity> Data Model` → `TASK-001-<entity>-data-model.md`

Every field in the canonical template (`docs/task-format.md`) is MANDATORY.
`N/A` is allowed only for genuinely inapplicable sections.

### Phase 5 — Context Initialization

If `.ai/stories/STORY-NNN-<slug>/context.md` is empty or missing, write:

```markdown
# STORY-NNN-<slug> Context

## Status

Tasks generated: <ISO date>
Tasks: TASK-001 … TASK-NNN

## Shared Artifacts

(Execution agents record file paths of artifacts consumed by multiple tasks.)

## Decisions

(Execution agents append decisions after each task.)

## Completed Tasks

(Execution agents mark tasks done here.)

## Open Questions

(Execution agents log blockers here.)
```

### Phase 6 — Summary

Print to the conversation:

```
STORY-NNN-<slug>: <title>
──────────────────────────────────────────────────────
TASK-001-<entity>-data-model      [no deps]        <Entity> Data Model
TASK-002-<entity>-types           [deps: 001]      <Entity> Types
TASK-003-<entity>-repository      [deps: 002]      <Entity> Repository
TASK-004-post-<entity>-route      [deps: 003]      POST /api/<entity> Route
TASK-005-<entity>-card-component  [deps: 002]      <Entity>Card Component
TASK-006-<entity>-page            [deps: 004,005]  /<entity> Page
TASK-007-<entity>-repo-tests      [deps: 003]      <Entity> Repository Unit Tests
──────────────────────────────────────────────────────
7 tasks written to .ai/stories/STORY-NNN-<slug>/tasks/
```

---

## Task Generation Heuristics

### Decomposition signals

| Story phrase | Generated concerns |
|---|---|
| "user can create an `<entity>`" | data model + types + service + POST route + form component + page |
| "user can view a list" | types + GET route + hook + list component + page |
| "user can edit" | PUT route + service.update method + edit form component |
| "user can delete" | DELETE route + service.delete method (often 1 task) |
| "store/persist/save X" | data model + migration |
| "search/filter" | GET route with query params + filter component (separate) |
| "authenticate" | auth middleware task + token util (always separate tasks) |
| "validate input" | validator util task (1 file, independent) |

### Naming conventions

```
TASK-NNN-<slug> — <Layer> <EntityName> <Action>

File name: TASK-NNN-<slug>.md  (used for the file on disk and all cross-references)
Heading:   TASK-NNN-<slug> — <Layer> <EntityName> <Action>

Slug rules:
  - Derived from the human title
  - Lowercase, hyphen-separated
  - 3–5 words; drop articles (a, an, the) and prepositions (for, of, in)

Examples:
  TASK-001-<entity>-data-model       — <Entity> Data Model
  TASK-002-<entity>-types            — <Entity> Types
  TASK-003-<entity>-repository       — <Entity> Repository
  TASK-004-post-<entity>-route       — POST /api/<entity> Route
  TASK-005-<entity>-card-component   — <Entity>Card Component
  TASK-006-use-<entity>-hook         — use<Entity> Hook
  TASK-007-<entity>-page             — /<entity> Page
  TASK-008-<entity>-repo-tests       — <Entity> Repository Unit Tests
```

### File path inference

Read `.ai/architecture.md § Path Conventions` for the full path pattern table.
Use those patterns when populating `## Allowed Files` in every task.

### New file creation

When a task introduces net-new files (not yet in the repo), the task MUST still
list them explicitly in `## Allowed Files` with the `[create]` marker. The
executor uses this list as its only write permission — omitting a new file from
the list causes the executor to skip creating it.

**When to mark `[create]`**

| Scenario | Files to mark |
|---|---|
| New data model added to schema | schema file — `[modify]`; migration file auto-generated, note it in Objective |
| New route segment | page file — `[create]`; add layout only if segment needs its own layout |
| New server action module | actions file — `[create]` |
| New repository module | repository file — `[create]` |
| New UI component | component file — `[create]` |
| New shared types file | types file — `[create]` |
| New custom hook | hook file — `[create]` |

**Directory scaffolding rule**: If a `[create]` file lives in a directory that
does not yet exist, include a note in the task Objective: "The executor MUST
create the parent directory `<dir>/` before writing the file." Do NOT list
directories as separate Allowed Files entries — only list the files themselves.

**`[create]` means**: the file does not exist yet; the executor must create it
from scratch with the exact content specified in Requirements. If the file
already exists when the executor runs, the executor MUST treat it as `[modify]`
and preserve existing exports.

### File deletion

If story analysis implies that a file should be deleted (e.g. replacing a
module, removing a deprecated route), **do NOT silently add it to Allowed Files
and do NOT proceed to write the task.**

Instead, stop and ask the user:

```
The story implies deleting `<exact/path/to/file.ts>`.
Reason: <one sentence why the deletion is needed>.
Should I include this deletion in the task? (yes / no / rename instead)
```

- If the user confirms **yes**: write the task with `[delete]` marker in
  Allowed Files and include a "Delete Steps" subsection in Requirements
  listing the exact shell command and any import cleanup.
- If the user says **no**: document the file as `[modify]` or exclude it.
- If the user says **rename**: use `[create]` for the new path and `[delete]`
  for the old path — two separate entries.

**Never infer that a deletion is safe.** Always surface it.

---

## Anti-Pattern Prevention

Run this checklist before writing any task file. Flag and fix violations.

See `docs/anti-patterns.md` for the full catalogue. Critical rules:

**ANTI-001 — Cross-domain task**: Files from ≥2 concern types in one task.
→ Split by concern.

**ANTI-002 — God task**: >5 allowed files.
→ Split by HTTP method, CRUD method group, or component.

**ANTI-003 — Vague objective**: Uses "refactor", "improve", "handle", "etc."
→ Name exact exports, signatures, field names.

**ANTI-004 — Missing file scope**: Allowed Files is empty or says "any relevant files".
→ List exact paths. Use `[TBD: dep TASK-NNN]` only with an explicit dependency.

**ANTI-005 — Implicit dependency**: Task imports from a file produced by another
task not listed in Dependencies.
→ Audit every import in Requirements; add missing deps.

**ANTI-006 — Non-verifiable criteria**: "works correctly", "looks good".
→ Every criterion must be a runnable command or specific assertion.

**ANTI-007 — Architecture drift**: Requirements say "choose appropriate pattern",
"use best practices", "feel free to add".
→ Prescribe the exact pattern, class name, error type.

**ANTI-008 — Missing context update**: Context Update section is empty.
→ Every task must contain the verbatim append block for `context.md`.

**ANTI-UI-001 — Raw value in UI task**: Requirements use hard-coded colors, sizes,
or other raw design values instead of design system tokens.
→ Replace with the token names defined in your stack (see `.ai/architecture.md § Stack Rules`).

**ANTI-UI-002 — Wrong primitive library**: Requirements reference a UI primitive
library not approved by the project stack.
→ Replace with the approved UI primitive library (see `.ai/architecture.md § Stack Rules`).

**ANTI-UI-003 — Missing class merging utility**: UI task has no mention of the
approved class merging utility.
→ Add the class merging utility requirement per `.ai/architecture.md § Stack Rules`.

---

## Mandatory Pre-Flight Checklist

Before writing any task file:

- [ ] Task touches a single concern type
- [ ] Task has ≤ 5 allowed files
- [ ] Objective names exact exports and file paths (no ambiguity)
- [ ] All file paths are exact (or `[TBD: dep TASK-NNN]` with explicit dep)
- [ ] All implicit upstream dependencies are listed in Dependencies
- [ ] ≥ 2 acceptance criteria that are mechanically verifiable
- [ ] Context Update section contains the verbatim append block
- [ ] Tasks are ordered so dependencies come first (lower ID = runs sooner)
- [ ] No banned phrases (see `docs/anti-patterns.md` § Red-Flag Phrase Blocklist)
- [ ] UI tasks include stack-approved token and class merging constraints

---

## Reference Files

- [docs/task-format.md](docs/task-format.md) — canonical template + field rules
- [docs/sizing-guide.md](docs/sizing-guide.md) — four gates + splitting patterns
- [docs/anti-patterns.md](docs/anti-patterns.md) — anti-pattern catalogue
- [examples/](examples/) — add your own project-specific task examples here
