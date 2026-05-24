# Review Workflow — Diff-Aware Playbook

This document expands the five-phase workflow defined in `SKILL.md` with
concrete commands, decision points, and stop conditions. Read-only throughout.

---

## Phase 1 — Scope discovery

Goal: know exactly what to review before reading a single line of code.

Commands (read-only):

```bash
git status                          # uncommitted state
git diff --stat main...HEAD         # branch changes vs main
git log --oneline main..HEAD        # commits on this branch
git diff --stat <base>...<head>     # user-specified range
```

Decision points:

| Signal                                  | Action                                          |
|-----------------------------------------|-------------------------------------------------|
| Branch tracks a known story (`STORY-NNN`) | Load the story doc read-only.                  |
| Commit references `TASK-NNN` / `QUICK-NNN` | Load that task/quick-task doc.                |
| Diff > 20 files OR > 1500 changed lines | Pause. Ask user: full review or scoped subset. |
| No diff vs `main`                       | Fall back to staged + unstaged working tree.   |
| Reviewing a PR by number                | `gh pr view N --json files,title,body`.        |

Stop condition: you have a finite, named list of files in scope and a known
acceptance criteria source (or explicit acknowledgement there is none).

---

## Phase 2 — Diff-aware reading

Read only what is necessary to judge the change.

Priority order:

1. The changed hunks themselves.
2. The 10–30 lines surrounding each hunk (existing context).
3. The function signature(s) the hunk depends on, if not visible in the hunk.
4. Call sites of changed exported symbols (only when regression risk is in
   question).
5. The matching test file for the changed module.

Avoid:

- Reading entire files when the hunk is self-contained.
- Following imports transitively beyond depth 1.
- Reading vendored / generated / lockfile changes line-by-line.

Parallel reads are encouraged when investigating independent files in the
same phase. Sequential reads when each read informs the next.

---

## Phase 3 — Analysis

Walk the axes from `SKILL.md § Phase 3`. For each finding, record:

- `path:line` (post-change line number)
- One-sentence problem statement
- One-sentence "why it matters" (impact / blast radius)
- One-sentence recommended direction (not code)
- Tentative severity (apply rubric)

If a finding requires reading outside the diff to confirm, do that read now,
then return. Do not let confirmation reads cascade into a full audit.

---

## Phase 4 — Acceptance criteria check

Source of acceptance criteria, in preference order:

1. The linked story/task document's "Acceptance Criteria" section.
2. The PR description's checklist.
3. Inferred from the task title + commit messages (mark as inferred).

For each criterion, one of:

- **Passed** — verified directly from the diff.
- **Failed** — directly contradicted by the diff or by a finding above.
- **Not verifiable from diff** — would need runtime, manual QA, or files
  outside the diff. State what would verify it.

Do not run the code. Do not execute tests. Verification is static, from the
diff.

---

## Phase 5 — Report

Emit the report exactly per `SKILL.md § Output Template`. Then stop.

Stop means:

- No "next steps" section.
- No offer to apply fixes.
- No follow-up tool calls that mutate state.
- Wait for the user's next instruction.
