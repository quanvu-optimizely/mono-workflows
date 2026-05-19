# Story File Format — Field Reference

Every story file written by `story-creator` MUST use this exact structure.
All nine sections are mandatory. Use `N/A` only for genuinely inapplicable fields.

---

## File Path

```
.ai/stories/STORY-NNN-<slug>/story.md
```

`NNN` is zero-padded to 3 digits. IDs are assigned in execution order:
stories with no upstream dependencies get the lowest IDs.
`<slug>` is a 3–5 word kebab-case summary derived from the story title
(e.g. `<entity>-data-layer`, `<feature>-ui`). Drop articles and prepositions.

---

## Canonical Template

```markdown
# STORY-NNN-<slug> — <Entity> <Concern> [Layer]

## Story Goal

One sentence. State the user-observable outcome this story delivers.
Must pass the "newspaper test": a non-engineer reading this sentence
understands what changes for the user.

## Business Context

2–5 sentences. Why does this story exist? What user need or product
requirement drives it? Reference the PRD section or feature name if known.
Do NOT describe implementation details here.

## Constraints

Hard boundaries the executor MUST respect. These are non-negotiable.

**Stack constraints:**
_(Copy the Stack Rules block verbatim from `.ai/architecture.md § Stack Rules`)_

**Scope constraints:**
- This story owns: <list the files/directories this story controls>
- This story MUST NOT modify: <list files/directories that are out of scope>
- Auth is out of scope unless this story is the Auth Setup story
- Global state management is out of scope unless explicitly in story goal

## Affected Areas

Exact file paths this story's tasks will create or modify.
Use `[TBD: determined by STORY-NNN]` only with an explicit dependency listed below.

**New files (created by this story's tasks):**
- `<exact/path/to/file.ts>` — <reason>
- `<exact/path/to/file.ts>` — <reason>

**Modified files:**
- `<exact/path/to/file.ts>` — <what changes>

## Technical Decisions

All architectural decisions the executor MUST follow. No deferred choices.
Each decision must be specific enough that two different executors arrive at
the same implementation.

**Data layer:**
- <exact model name, field names, field types>
- <exact migration strategy: additive-only fields, no destructive changes>

**Service layer:**
- <exact class/function names, method signatures, error types>

**API layer:**
- <exact HTTP method, route path, request/response shape>

**UI layer (if applicable):**
- <exact component name, props interface, styling approach>

## Acceptance Criteria

Markdown checklist. Each criterion must be verifiable by running a command
or inspecting a specific, named output. No subjective criteria.
Minimum 3 criteria.

- [ ] `<build-command>` exits with code 0 after all tasks complete
- [ ] <specific verifiable check — named export, curl command, etc.>
- [ ] <specific verifiable check>

## Risks

Known risks that could block execution or cause rework.

| Risk | Likelihood | Mitigation |
|---|---|---|
| <risk description> | Low / Med / High | <mitigation strategy> |

## Dependencies

Upstream stories that MUST be complete before this story's tasks begin.

- STORY-NNN — <title> — produces `<artifact>` consumed by this story

If none: `None`

## Execution Strategy

How the executor should approach this story's task chain.

**Entry task**: TASK-NNN (no dependencies — run first)

**Task chain:**
```
TASK-001  [no deps]      <task name>
  ↓
TASK-002  [deps: 001]    <task name>
  ↓
TASK-003  [deps: 002]    <task name>
  ↓
TASK-004  [deps: 002]    <task name>   ← can run parallel with TASK-003
```

**Parallelism notes:**
- <which tasks can run in parallel and why>

**Context update protocol:**
After each task completes, the executor appends a structured block to
`.ai/stories/STORY-NNN-<slug>/context.md`. See `templates/context.md` for the schema.
The next task executor MUST read context.md before starting.
```

---

## Field Rules

### `# STORY-NNN-<slug> — <title>`

- `NNN` is zero-padded (001, 002, … 099, 100).
- `<slug>`: 3–5 words, lowercase, hyphen-separated, derived from the title. Drop articles/prepositions.
- Title: `<Entity> <Concern> [Layer]` in title case.
- Entity vocabulary: read from `.ai/architecture.md § Domain`. Use names exactly as defined there.
- Concern vocabulary: `Data Layer`, `List View`, `Detail View`, `Edit Flow`,
  `Creation Flow`, `Search`, `Service`, `API`, `Auth Setup`.
- Keep under 60 characters.

### `## Story Goal`

- Exactly one sentence.
- Subject is always the user or system actor.
- States observable outcome, not implementation step.
- MUST NOT use: "implement", "create the schema", "add a repository", "refactor".
- MUST use: "user can", "system stores", "system returns", "user sees".

### `## Business Context`

- 2–5 sentences. Plain language.
- Answers: why does this exist? What product requirement drives it?
- MUST NOT describe implementation details (no mention of ORM, routes, components).
- MAY reference PRD section numbers or feature names.

### `## Constraints`

- Split into Stack constraints (always present, project-wide) and Scope constraints
  (story-specific — what this story owns and what it must not touch).
- Every story's stack constraints MUST be copied verbatim from `.ai/architecture.md § Stack Rules`.
- Scope constraints must be precise: name exact directories and files.
- NEVER write: "don't modify unrelated files" — that is not a constraint, it is noise.

### `## Affected Areas`

- One entry per file. Exact path from project root.
- Mark each `[new]` or `[modify]`.
- If path is unknown until upstream story completes: `[TBD: STORY-NNN-<slug>]`.
- Group by: New files / Modified files.
- NEVER write: "all related files", "src/**", "as appropriate".

### `## Technical Decisions`

- One decision per bullet. Specific enough to yield identical output from two executors.
- Name exact: model names, field names and types, interface names, HTTP method and
  route path, component prop names, error type names.
- MUST NOT defer: "choose the appropriate pattern", "use best practices",
  "implement as you see fit".
- If a decision is genuinely unknown, log it in Risks and set a spike task.

### `## Acceptance Criteria`

- Markdown checklist (`- [ ] …`). Minimum 3 items.
- Always include the build command (from `.ai/architecture.md § Commands`) as the first item.
- For stories with API tasks: include a `curl` command with expected status code.
- For stories with data tasks: include a named export or schema check.
- No subjective criteria ("looks correct", "is fully working", "is complete").

### `## Risks`

- Markdown table with columns: Risk, Likelihood, Mitigation.
- Include at minimum: type safety risks, missing dependency risks.
- If no risks: include one row: "No identified risks | Low | N/A".
- Do NOT leave the table empty.

### `## Dependencies`

- Every upstream `STORY-NNN-<slug>` whose output this story imports or modifies.
- Include what artifact the upstream story produces.
- If none: write `None` — do not omit the section.
- Implicit dependencies (shared types, repositories) MUST be listed.

### `## Execution Strategy`

- Must name the entry task (lowest ID, no dependencies).
- Must show the full task chain with dependency annotations.
- Must note any tasks that can run in parallel.
- Must include the context update protocol reminder (verbatim instruction to executor).
