# Dependency Graph & Parallel Execution

Task-creator emits an explicit **DAG (directed acyclic graph)** over the tasks
in a story. Each task carries structured metadata so downstream orchestrators
(Cline, executor fleets, schedulers) can:

- batch independent tasks into parallel groups
- respect hard ordering where outputs feed inputs
- detect resource conflicts before scheduling
- identify critical-path tasks and bottlenecks

The graph is generated in **Phase 2** of the workflow and validated in
**Phase 2.5 (Conflict Detection)** before any task file is written.

---

## Relationship Types

Every edge in the graph is labeled. The classification determines whether
two tasks may run in parallel.

| Type | Symbol | Meaning | Parallel-safe? |
|---|---|---|---|
| `hard` | `→` | Downstream task imports/consumes a concrete artifact (file, export, schema) produced by the upstream task. | NO — must serialize. |
| `soft` | `⇢` | Downstream task is logically *easier* if upstream runs first (shared conventions, established patterns) but does not import anything. | YES — may parallelize, with a sequencing preference. |
| `none` | — | No relationship. | YES. |
| `resource_conflict` | `⚠` | Two tasks touch the same file, same DB migration, same env var, or same global state. Not strictly dependent, but cannot run simultaneously without merge conflicts. | NO — serialize OR split files. |
| `sequencing_preference` | `↻` | Order is recommended for review ergonomics (e.g., land schema before service) but not technically required. | YES — preference is advisory only. |

`hard` is the only edge that creates a true dependency in `depends_on`.
`resource_conflict` populates `resource_conflicts` and forces serialization
via the orchestrator, but does NOT appear in `depends_on`.

---

## Dependency Metadata Schema

Every task file MUST include a `## Dependency Metadata` block with the
following YAML payload (verbatim keys; order preserved):

```yaml
task_id: TASK-NNN-<slug>
depends_on:           # hard dependencies only — upstream task IDs
  - TASK-NNN-<slug>
soft_deps: []         # soft sequencing preferences (advisory)
blocked_by: []        # tasks that MUST complete first for any reason
                      # (superset of depends_on + resource_conflicts)
parallelizable: true  # boolean; false if any hard dep or resource conflict
parallel_group: <group-name>   # logical batch label (see Group Naming)
resource_conflicts:   # tasks that touch overlapping resources
  - TASK-NNN-<slug>: <reason e.g. "same file: src/lib/auth.ts">
critical_path: false  # true if removing this task delays story completion
relationship_notes: |
  (optional 1–2 line free-text on non-obvious coupling)
```

### Field rules

- `depends_on` — populated **only** from `hard` edges. Empty list = leaf node.
- `soft_deps` — `soft` and `sequencing_preference` edges. Orchestrator may
  ignore for parallelization but should log them.
- `blocked_by` — derived. Union of `depends_on` and the task IDs in
  `resource_conflicts`. This is the field the scheduler reads.
- `parallelizable` — `false` if `depends_on` is non-empty for any task in
  the *same* parallel group, OR if `resource_conflicts` is non-empty.
- `parallel_group` — see Group Naming below. Tasks with the same group label
  are candidates for concurrent execution **after** all `blocked_by` clear.
- `resource_conflicts` — populated by the Conflict Detection pass.
- `critical_path` — task lies on the longest dependency chain to story
  completion. At least one task per story must be flagged.

---

## Group Naming

Parallel groups are coarse logical batches, NOT individual tasks. Use these
canonical names; invent new ones only if no canonical group fits.

| Group | Members |
|---|---|
| `data-foundation` | data model, migration, shared types |
| `data-access` | repositories, validators |
| `backend-logic` | services, server actions |
| `api-surface` | API routes (one HTTP method each) |
| `frontend-hooks` | custom hooks, client-side data fetchers |
| `frontend-ui` | components, pages |
| `tests-unit` | unit tests |
| `tests-integration` | integration / e2e tests |
| `docs` | documentation, READMEs |
| `infra` | env vars, config, build files |

Tasks in the **same group** that have no `depends_on` edge between them and
no `resource_conflicts` are parallel-safe.

---

## Conflict Detection Rules

Phase 2.5 runs before any task file is written. Two tasks have a
`resource_conflict` if ANY of the following hold:

1. **Same-file modification** — Both list the same path in Allowed Files
   with `[modify]` (or both `[create]` on the same path).
2. **Shared migration window** — Both produce DB migrations on the same
   schema. Schema mutations MUST serialize.
3. **Shared global state** — Both modify the same store/reducer, the same
   env-loading module, the same DI container registration, or the same
   global config object.
4. **Coupled domain** — Both touch tightly coupled modules (auth middleware
   + session handler, router + route registry, theme provider + token file).
5. **Infra collision** — Both modify the same `package.json`, lockfile,
   Dockerfile, or CI workflow.

When a conflict is detected, choose the **lightest** fix in this order:

1. Split files so each task owns disjoint paths.
2. Merge the two tasks if combined size still passes the four sizing gates.
3. Serialize: convert the weaker side to a `hard` dep on the stronger.

Never silently let a conflict survive into the generated tasks.

---

## Parallel-Safe Examples

```
TASK-005 — <Entity>Card Component        group: frontend-ui     deps: TASK-002
TASK-006 — POST /api/<entity> Route      group: api-surface     deps: TASK-004
```

Different files, different groups, no shared state → safe to run concurrently
once their respective `depends_on` clear.

```
TASK-010 — Repository Unit Tests         group: tests-unit      deps: TASK-003
TASK-011 — Page Component                group: frontend-ui     deps: TASK-005
```

Tests and UI never touch each other's files → parallel-safe.

---

## Unsafe Parallel Examples

```
TASK-005 — Refactor auth middleware       file: src/lib/auth.ts
TASK-006 — Update session token handling  file: src/lib/auth.ts
```

Same file → `resource_conflict`. Serialize or merge.

```
TASK-008 — Add users.email index          DB migration
TASK-009 — Drop users.legacy_id column    DB migration
```

Schema mutation collision → serialize.

```
TASK-012 — Update Redux auth slice
TASK-013 — Update Redux session slice
```

Even in separate files, the `store/index.ts` registry mutation forces a
`resource_conflict` on the registry. Serialize.

---

## DAG Output Format (Phase 6 Summary)

Phase 6 prints a textual DAG in addition to the task list:

```
Parallel Execution Plan
─────────────────────────────────────────────────────────────
Wave 1 (no deps):          TASK-001, TASK-002
Wave 2 (after Wave 1):     TASK-003 ‖ TASK-004
Wave 3 (after Wave 2):     TASK-005 ‖ TASK-006 ‖ TASK-007
Wave 4 (after Wave 3):     TASK-008

Critical path:             TASK-001 → TASK-003 → TASK-005 → TASK-008
Parallel groups:           data-foundation, backend-logic,
                           api-surface, frontend-ui, tests-unit
Resource conflicts:        none
Bottlenecks:               TASK-003 (4 downstream consumers)
```

`‖` means "may run in parallel". Each wave is gated by completion of the
previous wave.

---

## Orchestration Guidance

Downstream orchestrators (Cline batch runners, fleet schedulers) consume
the `## Dependency Metadata` blocks and SHOULD:

1. Topologically sort by `blocked_by`.
2. Within each topological wave, group by `parallel_group`.
3. Within each group, dispatch tasks concurrently up to the configured
   worker pool.
4. Treat `critical_path: true` tasks as high-priority — schedule them on
   the fastest available worker.
5. Surface `soft_deps` as informational ordering hints in logs only.
6. Refuse to schedule any task whose `resource_conflicts` overlap with a
   currently-running task.

---

## Best Practices

- **Prefer explicit hard deps over implicit assumptions.** If a task reads
  an export, the producing task belongs in `depends_on`.
- **Maximize safe parallelism.** Tasks that share no files and no global
  state SHOULD be parallelizable — flag them so.
- **Minimize coordination.** A task with > 3 entries in `blocked_by` is a
  bottleneck candidate — consider extracting a leaf prerequisite to flatten.
- **Detect hidden coupling.** Watch for shared registries, DI containers,
  shared barrel exports, and middleware chains.
- **Keep modeling deterministic.** Two runs of task-creator on the same
  story MUST produce identical `depends_on` and `parallel_group` values.
- **Avoid over-serialization.** Default to `parallelizable: true` and only
  reduce when a real conflict exists.
