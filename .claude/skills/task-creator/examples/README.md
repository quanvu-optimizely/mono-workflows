# Task Examples

This directory is for project-specific task examples.

Add one or two real completed tasks here so future sessions have concrete
references for the correct format and level of detail.

## Recommended examples to add

- `TASK-001-example.md` — A leaf-node data model task (first task in a story, no deps)
  - `depends_on: []`, `parallel_group: data-foundation`, `critical_path: true`
- `TASK-004-example.md` — An API route task (depends on service and types)
  - `depends_on: [TASK-002, TASK-003]`, `parallel_group: api-surface`
- `TASK-005-parallel-safe-example.md` — A UI component task that runs in
  parallel with an unrelated API route task
  - `parallelizable: true`, `resource_conflicts: []`, different `parallel_group`
- `TASK-006-resource-conflict-example.md` — A task that touches the same
  file as another task in the same story
  - `parallelizable: false`, `resource_conflicts: [TASK-007: "same file: src/lib/auth.ts"]`

## Dependency graph patterns

Each example should illustrate one DAG pattern from
[`../docs/dependency-graph.md`](../docs/dependency-graph.md):

1. **Sequential chain** — schema → repository → service → route
2. **Fan-out** — types task feeds many independent consumers
3. **Fan-in** — page depends on both API and component branches
4. **Parallel-safe pair** — UI and backend tasks running concurrently
5. **Conflict serialization** — two tasks forced to serialize via
   `resource_conflicts`

## Why examples matter

The `task-creator` skill reads these files to calibrate output for your specific
stack, naming conventions, and path patterns. Generic examples are less useful
than real ones from your project.

## Format

Copy from a completed task in `.ai/stories/STORY-NNN-<slug>/tasks/` — pick tasks
where the executor followed the format precisely and the output was correct.
