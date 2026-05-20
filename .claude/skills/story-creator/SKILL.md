---
name: story-creator
description: >
  Transform a product requirement or feature request into bounded implementation
  stories, then orchestrate task generation via task-creator. Triggers on
  /story-creator command or when the user asks to "create a story for X",
  "break this feature into stories", or "plan implementation for Y". Produces
  architecture-safe, independently executable stories with full task chains.
version: 1.0.0
---

# story-creator — Implementation Story Generator

Transforms a product requirement into bounded, architecture-safe implementation
stories, then invokes `task-creator` to produce executable task chains for Cline.

**You are operating in architect-planner mode. You write story files and task
files only. You MUST NOT write or modify production source code outside `.ai/`.**

---

## Two-Agent System Architecture

This skill operates within a strict two-role model:

| Role | Tool | Responsibilities |
|---|---|---|
| **Architect-Planner** | Claude Code | Story creation, architecture decisions, task orchestration |
| **Executor** | Cline | Task implementation, context updates, bounded file writes |

**Separation rules (non-negotiable):**

- Claude Code writes ONLY to `.ai/` (stories, tasks, architecture, prompts).
- Cline writes ONLY to files listed in `Allowed Files` of the active task.
- Cline NEVER makes architecture decisions — it appends decisions to `context.md`.
- Claude Code NEVER writes production code — it defines what Cline should write.

---

## Stack Reference

Read `.ai/architecture.md § Stack` and `.ai/architecture.md § Stack Rules`.

Translate those rules into per-story constraints. Do not leave them implicit.
Every story's `## Constraints` section MUST copy the Stack Rules block verbatim
from `.ai/architecture.md`.

---

## Domain Vocabulary

Read `.ai/architecture.md § Domain`. Use entity names exactly as defined there
in story titles, constraint blocks, and affected areas.

---

## Invocation Patterns

```
/story-creator "Add search to the notes list"
/story-creator --prd path/to/requirements.md
/story-creator --prd path/to/requirements.md --dry-run
/story-creator --tier=medium "Add archived filter"          # force a planning tier
/story-creator STORY-001-<slug> --tasks                     # generate tasks for an existing story
/story-creator STORY-001-<slug> --retask                    # regenerate tasks (preserves story.md)
```

| Flag | Behavior |
|---|---|
| `"<description>"` | Inline requirement; auto-assign STORY-NNN-<slug> |
| `--prd <path>` | Read requirement from a markdown file |
| `--tier=<name>` | Force planning tier (trivial / medium / large / epic). Bypasses tier-classifier. |
| `--dry-run` | Print story plan to conversation; do NOT write files |
| `STORY-NNN-<slug> --tasks` | Skip story creation; invoke task-creator only |
| `STORY-NNN-<slug> --retask` | Re-run task-creator; existing tasks are overwritten |

---

## Workflow

### Phase 0 — Locate and Parse Input

1. **Read `.ai/architecture.md`** — load stack constraints, domain vocabulary,
   path conventions, and open architecture decisions. Stop if the file is
   missing and tell the user to create it.
2. Identify input source: inline description, PRD file, or existing STORY-XXX.
3. If `--prd <path>`: read the file; extract feature name, goals, user personas,
   constraints, out-of-scope items. Stop if file is missing.
4. If inline description: treat as a one-sentence feature statement.
5. Scan `.ai/stories/` for highest `STORY-NNN` and assign the next ID. Generate a
   kebab-case slug from the story title (3–5 words, lowercase, drop articles and
   prepositions). Full directory name: `STORY-NNN-<slug>`
   (e.g. `STORY-001-<entity>-data-layer`, `STORY-004-<feature>-ui`).
6. Honor any open architecture decisions recorded in `.ai/architecture.md § Architecture Decisions`.

### Phase 0.5 — Planning Tier Classification

Before doing feature analysis, decide **how much planning this request deserves**.

1. Read `.ai/planning-tiers.md` (single source of truth for tier behavior).
2. If the user passed `--tier=<name>`, use that tier and skip to the tier-branch.
3. Otherwise invoke the `tier-classifier` skill on the input (inline description
   or PRD body). It returns a tier + rationale block.
4. Record `Tier: <name>` and `Tier Rationale: <one line>` at the top of every
   artifact this run produces (quick task, story metadata, or epic.md).

**Tier branch** (each branch is described in `docs/planning-tiers.md`):

| Tier      | What this run produces                                                    | Phases below that run                                            |
|-----------|---------------------------------------------------------------------------|------------------------------------------------------------------|
| `trivial` | One quick-task file in `.ai/quick-tasks/QUICK-NNN-<slug>.md`              | Skip Phases 1–6. Use the Quick Task path in `docs/planning-tiers.md`. |
| `medium`  | One slim story (4 required sections) with ≤ 3 tasks                       | Run Phase 1 lite, skip Phase 2 split, run a slim Phase 4, Phase 5, Phase 6. |
| `large`   | Default. One full story (9 sections) with ≤ 7 tasks                       | Run Phases 1–6 as written below.                                 |
| `epic`    | Epic dir `.ai/epics/EPIC-NNN-<slug>/` with phased roadmap + Phase-1 stories | Run Phase 1, then the Epic path in `docs/planning-tiers.md`, then Phase 5/6 per generated story. |

Do **not** duplicate signal weights, thresholds, or behaviors inside this file —
they live in `.ai/planning-tiers.md` and `docs/planning-tiers.md`.

### Phase 1 — Feature Analysis

Decompose the input into structured dimensions:

```
DOMAIN ANALYSIS
  ├── Primary domain (primary entity from .ai/architecture.md § Domain)
  ├── Secondary domains touched
  ├── Layer span (data → service → API → UI)
  └── Cross-cutting concerns (auth, validation, error handling)

DELIVERY DIMENSIONS
  ├── User-visible outcomes (what changes for the user)
  ├── Data changes (new models, modified fields)
  ├── API surface changes (new routes, modified contracts)
  └── UI changes (new components, modified pages)
```

Output: a plain-language feature summary (3–6 bullets). This becomes the
Business Context in the story.

### Phase 2 — Story Boundary Definition

Apply the Story Boundary Rules to carve the feature into stories:

**One story = one coherent business/domain concern.**

Boundary rules:

- A story MUST NOT span more than one primary entity's data layer.
- A story MUST NOT mix API implementation with UI implementation.
- A story CAN include types, repository, service, and API for the same entity.
- A story CAN include a hook and its corresponding UI component if they are
  tightly coupled and the total task count stays ≤ 7.
- Auth, validation, and error utilities MUST each be their own story if they
  introduce new shared modules.
- Split by layer when a story would produce > 7 tasks after decomposition.

Produce a story list. Example:

```
STORY-001-<entity>-data-layer     <Entity> Data Layer Foundation     (schema + types + repository + service)
STORY-002-<entity>-api            <Entity> CRUD API                  (GET + POST + PUT + DELETE routes)
STORY-003-<entity>-list-view      <Entity> List View                 (hook + list component + page)
STORY-004-<feature>-ui            <Feature> UI                       (feature hook + feature component)
```

### Phase 3 — Story Sizing Check

For each story, run all five gates. Fail any gate → split the story.

```
TASK COUNT      : ≤ 7 tasks after decomposition
DOMAIN COUNT    : = 1 primary entity
LAYER SPAN      : ≤ full stack (data → UI) for one entity
EXECUTOR RISK   : no decision requires real-time architecture judgment
DEPENDENCY DEPTH: ≤ 2 upstream stories
```

See `docs/sizing-guide.md` for split patterns and merge conditions.

### Phase 4 — Story File Generation

For each story, write `.ai/stories/STORY-NNN-<slug>/story.md` using the canonical
template (`docs/story-format.md`). All nine sections are MANDATORY.

Initialize `.ai/stories/STORY-NNN-<slug>/context.md` from `templates/context.md`.
Create `.ai/stories/STORY-NNN-<slug>/tasks/` (empty directory — task-creator fills it).
Create `.ai/stories/STORY-NNN-<slug>/logs/` (empty directory — executor fills it).

### Phase 5 — Task Orchestration via task-creator

For each story (or the single story if invoked for one), invoke `task-creator`:

```
INVOCATION PROTOCOL:

1. Read the completed story.md.
2. Call: /task-creator STORY-NNN-<slug>
   (task-creator reads story.md and context.md, generates tasks/)
3. After task-creator completes, verify:
   - All tasks in tasks/ have sequential IDs.
   - No task violates its own constraints.
   - Dependency chain is acyclic (no circular deps).
4. Record task index in context.md under "## Task Index".
```

**Do NOT generate tasks manually.** Always delegate to task-creator.
This enforces a single canonical task format across the project.

See `docs/orchestration-protocol.md` for the full integration spec.

### Phase 6 — Summary

Print to the conversation:

```
STORY PLAN
══════════════════════════════════════════════════════════════
STORY-001-<entity>-data-layer  <Entity> Data Layer Foundation
  Tasks: TASK-001 … TASK-005  (5 tasks)
  Domains: <Entity>
  Entry point: TASK-001 (no deps)

STORY-002-<entity>-list-view  <Entity> List View
  Tasks: TASK-006 … TASK-010  (5 tasks)
  Domains: <Entity>
  Entry point: TASK-006 (deps: STORY-001 complete)
══════════════════════════════════════════════════════════════
2 stories written to .ai/stories/
Next: run /task-creator STORY-001-<entity>-data-layer to generate implementation tasks.
```

---

## Story Generation Heuristics

### Feature → Story decomposition signals

| Feature phrase | Story split |
|---|---|
| "user can create/view/edit/delete a `<entity>`" | `<Entity>` Data Layer + `<Entity>` UI (2 stories) |
| "user can upload/record `<media>`" | `<Media>` Data Layer + `<Media>` UI (2 stories) |
| "process/transform data automatically" | Processing Data Layer + Processing Service (2 stories) |
| "user can tag/label `<entity>`" | Tag Data Layer + `<Entity>`-Tag Association (1–2 stories) |
| "user can search `<entity>`" | Search API + Search UI (2 stories) |
| "user logs in / authenticates" | Auth Setup (1 story, always isolated) |
| "validate input" | Validator utility (part of domain story, not standalone unless shared) |
| "dashboard / home page" | Dashboard UI (1 story, depends on entity layers) |

### Story naming convention

```
STORY-NNN-<slug>  <Entity> <Concern> [Layer]

Directory name: STORY-NNN-<slug>  (used for .ai/stories/ folder and all cross-references)
Human title:    <Entity> <Concern> [Layer]  (used inside story.md heading after the em dash)

Slug rules:
  - Derived from the human title
  - Lowercase, hyphen-separated
  - 3–5 words; drop articles (a, an, the) and prepositions (for, of, in)

Examples:
  STORY-001-<entity>-data-layer        <Entity> Data Layer Foundation
  STORY-002-<entity>-list-view         <Entity> List View
  STORY-003-<entity>-detail-view       <Entity> Detail View
  STORY-004-<entity>-search-api        <Entity> Search API
  STORY-005-<entity>-search-ui         <Entity> Search UI
  STORY-006-<feature>-service          <Feature> Service
  STORY-007-<entity2>-data-layer       <Entity2> Data Layer
  STORY-008-user-auth-setup            User Auth Setup
```

### Affected areas — path conventions

Read `.ai/architecture.md § Path Conventions` for the full path pattern table.
Use those patterns to populate the `## Affected Areas` section of every story.

---

## Anti-Pattern Prevention

Run this checklist before writing any story file. Flag and fix violations.

See `docs/anti-patterns.md` for the full catalogue.

**SANTI-001 — Giant story**: Story decomposes into > 7 tasks.
→ Split by layer: data layer story + UI story.

**SANTI-002 — Vague goal**: Goal uses "improve", "enhance", "refactor", "handle X better".
→ Rewrite as a user-observable outcome: "User can create an `<entity>` with a title and body."

**SANTI-003 — Cross-domain refactor**: Story touches 2+ primary entities' data layers.
→ One story per primary entity. Shared concerns go in a dedicated utility story.

**SANTI-004 — Hidden dependency**: Story claims no upstream dependencies but references
types/models from another story that hasn't been written yet.
→ List the upstream story in Dependencies. Set execution order accordingly.

**SANTI-005 — Architecture drift**: Technical Decisions section defers choices to executor
("use whichever pattern fits", "choose the best approach").
→ Every architectural decision MUST be made in the story by the planner.

**SANTI-006 — Unconstrained executor scope**: Affected Areas is vague ("all related files").
→ List exact file paths. Use `[TBD: determined by STORY-NNN]` only with explicit dependency.

**SANTI-007 — Global context explosion**: Story's context.md grows unbounded because
executor appends raw diffs instead of structured decisions.
→ Enforce the context.md append schema (see templates/context.md).

**SANTI-008 — Missing acceptance criteria**: Story has no story-level criteria.
→ Every story needs ≥ 3 acceptance criteria verifiable via the build command or curl.

**SANTI-009 — Premature UI**: Story includes UI tasks before its data layer story is complete.
→ UI stories MUST list the data layer story as a dependency.

**SANTI-010 — Auth/validation coupling**: Auth or validation logic is embedded in a domain story.
→ Auth and shared validators are always isolated stories or tasks.

---

## Mandatory Pre-Flight Checklist

Before writing any story file:

- [ ] Story represents one coherent business/domain concern
- [ ] Story decomposes into ≤ 7 tasks
- [ ] Story has exactly one primary entity
- [ ] Story goal is user-observable (not implementation-focused)
- [ ] All Technical Decisions are made (no deferred choices)
- [ ] Affected Areas list exact file paths (or `[TBD: STORY-NNN]`)
- [ ] All upstream story dependencies are listed
- [ ] ≥ 3 acceptance criteria that are mechanically verifiable
- [ ] No banned phrases (see `docs/anti-patterns.md`)
- [ ] Execution strategy names the entry task and dependency chain
- [ ] context.md template is initialized

---

## Reference Files

- [docs/planning-tiers.md](docs/planning-tiers.md) — tier branches: Quick Task, Slim Story, Full Story, Epic
- [docs/story-format.md](docs/story-format.md) — canonical 9-section story template
- [docs/sizing-guide.md](docs/sizing-guide.md) — story sizing gates and split patterns
- [docs/anti-patterns.md](docs/anti-patterns.md) — 10 story anti-patterns + banned phrases
- [docs/orchestration-protocol.md](docs/orchestration-protocol.md) — task-creator integration spec
- [.ai/planning-tiers.md](../../../.ai/planning-tiers.md) — central tier configuration (signals, thresholds, behaviors)
- [templates/story.md](templates/story.md) — blank story template
- [templates/context.md](templates/context.md) — blank context template
- [examples/](examples/) — add your own project-specific story examples here
