# Task File Format — Field Reference

Every task file written by `task-creator` MUST use this exact structure.
All sections are mandatory. Use `N/A` only for genuinely inapplicable fields.

---

## File Path

```
.ai/stories/STORY-NNN-<slug>/tasks/TASK-NNN-<slug>.md
```

`NNN` is zero-padded to 3 digits. IDs are assigned in dependency order:
leaf nodes (no deps) get the lowest IDs.
`<slug>` is a 3–5 word kebab-case summary derived from the task title
(e.g. `<entity>-data-model`, `post-<entity>-route`). Drop articles and prepositions.

---

## Canonical Template

```markdown
# TASK-NNN-<slug> — <Layer> <Entity> <Action>

## Objective

One paragraph, 2–5 sentences. State exactly what the execution agent must
produce: file names, exported symbols, function signatures, observable
side-effects. Must be specific enough that a different agent produces the
same output. No optionality. No ambiguity.

## Allowed Files

The execution agent MUST only create or modify these files.
Any write outside this list is a constraint violation.

- `path/to/file.ts` — [create|modify] — brief reason
- `path/to/other.ts` — [create|modify] — brief reason

## Forbidden

Files and directories the execution agent MUST NOT touch.

- `<data-layer-dir>/` — data layer is out of scope for this task
- `<schema-file>` — schema is owned by TASK-NNN-<slug>

Always end with:
- All files not listed in Allowed Files

## Requirements

Numbered list. Each item is a single, actionable, testable requirement.
Use MUST / MUST NOT (RFC 2119). Name exact exports, interface fields,
function signatures. Never defer decisions to the executor.

1. ...
2. ...
3. ...

## Acceptance Criteria

Markdown checklist. Each criterion must be verifiable by running a command
or inspecting a specific, named output. No subjective criteria.

- [ ] `<build-command>` exits with code 0
- [ ] `<specific verifiable check>`
- [ ] `<specific verifiable check>`

## Dependencies

Upstream tasks that MUST be completed before this task starts.
List each with the artifact it produces.

- TASK-NNN-<slug> — produces `<artifact>` consumed by this task

If none: `None`

## Validation Steps

Ordered list of shell commands runnable from the project root.
Each command includes a comment explaining what it verifies.

1. `<build-command>` — confirms compilation is clean
2. `<command>` — <what it verifies>
3. `<command>` — <what it verifies>

## Context Update

After completing this task, the execution agent MUST append the
following block to `.ai/stories/STORY-NNN-<slug>/context.md` under
"Completed Tasks":

\`\`\`
## TASK-NNN-<slug> — <name> (completed <YYYY-MM-DD>)

### Decisions
- <any deviations from the plan and the reason>

### Files Changed
- `path/to/file.ts` — created
- `path/to/other.ts` — modified

### Notes
- <anything the next task executor should know>
\`\`\`
```

---

## Field Rules

### `# TASK-NNN-<slug> — <title>`

- `NNN` is zero-padded (001, 002, … 099, 100).
- `<slug>`: 3–5 words, lowercase, hyphen-separated, derived from the title. Drop articles/prepositions.
- Title: `<Layer> <EntityName> <Action>` in title case.
- Layer vocabulary: `Data Model`, `Migration`, `Types`, `Repository`, `Service`,
  `Validator`, `API Route`, `Server Action`, `Hook`, `Component`, `Page`,
  `Tests`, `Config`.
- Keep under 60 characters.

### `## Objective`

- 2–5 sentences. Single paragraph.
- MUST name: exact output file(s), exported symbol names, key method signatures.
- MUST NOT use: "refactor", "improve", "handle", "manage", "deal with",
  "etc.", "as needed", "properly", "correctly".
- MUST NOT defer architectural decisions to the executor.

### `## Allowed Files`

- One entry per file, with exact path from project root.
- Mark each `[create]`, `[modify]`, or `[delete]`.
  - `[create]` — file does not yet exist; executor must create it. If the file
    exists at run time, executor treats it as `[modify]` and preserves exports.
  - `[modify]` — file already exists; executor edits only the relevant section.
  - `[delete]` — file must be removed. **Only present after explicit user confirmation during task planning.** Executor MUST ask the user again before deleting.
- If a new file lives in a directory that does not yet exist, add a note in
  the Objective: "Executor MUST create parent directory `<dir>/` before writing
  the file." Do not list directories as separate entries.
- If a path cannot be determined until a dependency runs, write:
  `[TBD: path determined by TASK-NNN-<slug> output]`
  and add TASK-NNN-<slug> to the Dependencies section.
- NEVER write: "any file in X", "relevant files", "src/**", "as needed".

### `## Forbidden`

- At minimum, list directories that would tempt an executor to drift.
- Always include: `All files not listed in Allowed Files`.
- For UI tasks, explicitly forbid data layer directories and API route directories.
- For data model tasks, explicitly forbid UI component and page directories.

### `## Requirements`

- 3–10 items. One actionable requirement per item.
- Use MUST / MUST NOT for hard constraints.
- Name exact: class names, function names, parameter types, return types,
  interface field names, error types thrown.
- For UI components: MUST specify the class merging utility and design token
  names per `.ai/architecture.md § Stack Rules` — never raw colors or values.
- Do NOT say "use appropriate error handling" — say "wrap in try/catch;
  throw `new AppError('DB_ERROR', message)` on failure".

### `## Acceptance Criteria`

- Markdown checklist (`- [ ] …`).
- Minimum 2 criteria; aim for 3–5.
- MUST include the build command (from `.ai/architecture.md § Commands`) as first criterion on every task.
- For API tasks: include a `curl` command with expected status code.
- For repository/service tasks: include a named export check or unit test.
- No subjective criteria ("looks correct", "works", "is implemented").

### `## Dependencies`

- Every upstream TASK-ID whose output this task imports or modifies.
- Include what artifact the upstream task produces.
- If none: write `None` — do not omit the section.
- Implicit imports (types, repositories) MUST be listed here.

### `## Validation Steps`

- Ordered numbered list of shell commands.
- Runnable from project root without modification.
- No placeholder commands. Every command must work on the current stack.
- Use commands from `.ai/architecture.md § Commands`.

### `## Context Update`

- Contains the verbatim block the execution agent copies into `context.md`.
- MUST reference the correct `STORY-NNN-<slug>` in the append path.
- Context Update heading MUST use the full `TASK-NNN-<slug>` identifier.
- MUST include `### Decisions`, `### Files Changed`, `### Notes` subsections.
- This section must NEVER be empty.
