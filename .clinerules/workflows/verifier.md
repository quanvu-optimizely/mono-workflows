# Verifier Workflow

**For**: Cline execution agent
**Triggered by**: Completion of an implementation task, before marking the task done
**Role boundary**: Verify the diff Cline just produced. Do not plan, redesign, or expand scope.

This is an execution-phase quality gate that runs **after** implementation and **before** task completion. Focus: build integrity, lint, basic runtime sanity, and acceptance criteria.

Out of scope: test execution, security audits, performance, architecture review.

---

## Quick Reference

```
Phase 1 → Analyze diff (changed files only)
Phase 2 → Build + lint checks
Phase 3 → Acceptance criteria + scope check
Phase 4 → Safe auto-fix + re-verify
Phase 5 → Report
```

Stop after Phase 5. Do not commit. Do not continue to next task.

---

## Severity

| Severity   | Meaning                                                                   | Action                                     |
|------------|---------------------------------------------------------------------------|--------------------------------------------|
| `critical` | Build/lint fails, scope violation, or acceptance criterion not satisfied. | STOP. Report. No auto-fix without user OK. |
| `major`    | Clear bug risk in changed code (null deref, missing await, dead branch).  | Auto-fix only if one-line obvious. Else report. |
| `minor`    | Dead import, stray log, trivial style.                                    | Auto-fix if trivial. Else report.          |

Tie-breaker: pick higher severity.

---

## Stopping Conditions

Stop immediately and produce the report when:

- Any `critical` finding.
- Build or lint fails and root cause is not a one-line trivial fix.
- Diff touches files outside the task's Allowed Files.
- Two auto-fix attempts on the same finding fail.

---

## Phase 1 — Diff Analysis

1. `git diff --name-only` against task base → list of changed files.
2. For each, read only the diff hunks.
3. Cross-check against task `Allowed Files`:
   - Changed but not allowed → `critical` scope violation.
   - Allowed but not changed → `info` note in report.

Do not scan the full repo.

---

## Phase 2 — Build + Lint

Run only the tools the project actually configures. Skip anything not configured (note "tool not configured" in the report). Do not invent commands.

In order:

1. **Typecheck** (e.g. `tsc --noEmit`, `mypy`, `cargo check`).
2. **Lint** (e.g. `eslint`, `ruff`, `clippy`), scoped to changed files when supported.
3. **Compile/build** (e.g. `npm run build`, `cargo build`) — only if the diff plausibly affects build output. Skip for doc-only diffs.
4. **Import resolution**: every new `import` / `require` resolves. Missing module → `critical`.
5. **Syntax**: any parse error → `critical`.

While reading diff hunks, also flag obvious runtime risks in **new** code only:

- New property access on values that may be `null`/`undefined` without a guard → `major`.
- Missing `await` on async calls whose result is used → `major`.
- Floating promises in code paths that should propagate errors → `major`.
- Swallowed errors (empty `catch`) → `major`.

Cite `path:line`. One line per finding.

---

## Phase 3 — Acceptance Criteria + Scope

For each acceptance criterion in the task file:

1. Point to the `path:line` (or function) that satisfies it.
2. Cannot point to one → criterion `unsatisfied` → `critical`.
3. Placeholders in new code (`TODO`, `FIXME`, `throw new Error("not implemented")`, empty bodies that should do work) not explicitly allowed by the task → `critical`.

Confirm every file the task said would change actually changed.

---

## Phase 4 — Safe Auto-Fix + Re-Verify

Auto-fix **only** when all hold:

1. Severity is `minor`, or `major` with a one-line obvious fix.
2. Fix stays inside the task's Allowed Files.
3. No signature, exported API, or schema change.
4. No new helpers, files, or abstractions.
5. Edit is local to the flagged line.

Allowed examples:

- Add missing `await`.
- Add a null guard right before a flagged dereference.
- Remove a dead import.
- Remove a stray `console.log` left by the implementer.

Forbidden even if "better":

- Rename variables across a file.
- Extract a helper.
- Reorder/reformat unrelated lines.
- Touch files outside Allowed Files.
- Add new tests or new modules.

After any auto-fix, re-run Phase 2 on the affected files. If a previously-passing check now fails, revert the fix and downgrade the finding to "reported, not fixed." Max two auto-fix attempts per finding.

Record every auto-fix in the report with `path:line` and a one-line description.

---

## Phase 5 — Report

Output exactly this block. No prose before or after.

```
# Verification Report

## Verified Areas
- Build: <pass | fail | n/a>
- Types: <pass | fail | n/a>
- Lint: <pass | fail | n/a>
- Acceptance Criteria: <N/M satisfied>

## Issues Found

### Critical
- <path:line> — <one-line problem>. <one-line proposed action>.

### Major
- <path:line> — <one-line problem>. <one-line proposed action>.

### Minor
- <path:line> — <one-line problem>.

## Auto-Fixed Issues
- <path:line> — <what changed>.

## Final Verification Status
<VERIFIED | NEEDS_ATTENTION>
```

Rules:

- One line per finding. Always include `path:line`.
- Empty section → write `- none`.
- `VERIFIED` requires: zero `critical`, zero `major`, all acceptance criteria satisfied, all configured checks passing.
- Anything else → `NEEDS_ATTENTION`.

---

## Forbidden Actions

Verifier MUST NOT:

- Modify files outside Allowed Files.
- Refactor, rename, or restructure beyond a one-line auto-fix.
- Add new modules, helpers, abstractions, or dependencies.
- Rewrite stable working code.
- Change public APIs, signatures, schemas, or config shapes.
- Run tests, security scans, or performance checks (out of scope for this workflow).
- Commit, push, open PRs, or mark the task complete.
- Continue to the next task.
- Run full-repo builds or scans when diff-scoped is enough.

---

## Token Efficiency

- Start from `git diff`. Never walk the file tree.
- Load full files only when a hunk is ambiguous in isolation.
- Skip phases with no surface area (doc-only diff → skip Phase 2 build/lint where irrelevant).
- Collapse repeated findings: same pattern 5+ times → one entry with a count and the `path:line` list.

---

## Good vs Bad Behavior

**Good**:

- Runs `tsc --noEmit` once, sees one error, adds the missing `await` on the flagged line, re-runs typecheck, reports the auto-fix, stops.
- Detects 2/3 acceptance criteria satisfied. Marks the third `critical` with the exact criterion text. Status `NEEDS_ATTENTION`. Stops.
- Diff touches `src/server/auth.ts` not in Allowed Files. Flags `critical` scope violation. Does not revert. Reports and stops.

**Bad**:

- Opens 30 unrelated files to "get a feel for the codebase."
- Renames `data` → `userRecord` across the file because it reads better.
- Adds a new `utils/validation.ts` helper to clean up a flagged null check.
- Re-runs the full project build for a docs-only change.
- Marks `VERIFIED` with one acceptance criterion unsatisfied because "it's basically done."
- Auto-fixes a `major` issue that requires a signature change.

---

## Output Templates

### All Clear

```
# Verification Report

## Verified Areas
- Build: pass
- Types: pass
- Lint: pass
- Acceptance Criteria: 4/4 satisfied

## Issues Found

### Critical
- none

### Major
- none

### Minor
- none

## Auto-Fixed Issues
- none

## Final Verification Status
VERIFIED
```

### Needs Attention

```
# Verification Report

## Verified Areas
- Build: pass
- Types: fail
- Lint: pass
- Acceptance Criteria: 2/3 satisfied

## Issues Found

### Critical
- src/api/orders.ts:88 — acceptance criterion "reject orders with negative quantity" not implemented. Add guard before persistence.
- src/api/orders.ts:14 — type error: `OrderInput.quantity` is `number | undefined`, used as `number`. Narrow before use.

### Major
- src/api/orders.ts:101 — floating promise on audit log write. Await or attach .catch.

### Minor
- src/api/orders.ts:3 — unused import `formatDate`.

## Auto-Fixed Issues
- src/api/orders.ts:3 — removed unused import `formatDate`.

## Final Verification Status
NEEDS_ATTENTION
```
