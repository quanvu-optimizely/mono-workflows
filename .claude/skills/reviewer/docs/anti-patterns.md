# Anti-Patterns — What the reviewer skill MUST NOT do

These are the failure modes that defeat the purpose of a read-only reviewer.
If you catch yourself doing any of these, stop and re-anchor to `SKILL.md`.

---

## 1. Auto-fixing during review

❌ Calling `Edit`/`Write` on a reviewed file because "the fix is one-line."
✅ Report the finding. Stop. Wait for the user.

The user invoked `/reviewer`, not `/quick-task`. Editing violates the
contract even if the edit is correct.

---

## 2. Emitting a "Suggested patch" code block

❌ Including a `diff` block or a "here's the fixed version" code block.
✅ Describe the fix direction in prose, one line.

Short illustrative snippets to *explain* a finding are fine (e.g. "uses `==`
where `===` is expected"). Whole-function replacements are not.

---

## 3. Mutating git state

❌ `git commit`, `git stash`, `git apply`, `git reset`, `git checkout --`,
   `git restore`.
✅ Only read-only git: `status`, `diff`, `log`, `show`, `blame`.

---

## 4. Scope creep into an unrequested repo audit

❌ "While I was there I also reviewed the entire `/utils` directory."
✅ Stay inside the diff. Mention adjacent risk once under Suggestions if
   genuinely load-bearing. Move on.

---

## 5. Reviewing the whole repo on a small change

❌ Reading 60 files because the PR touches imports across the codebase.
✅ Read changed hunks. Pull in additional files only when needed to
   adjudicate a specific finding.

---

## 6. Style-nit avalanche

❌ Listing 30 formatting/whitespace/import-order findings.
✅ Drop them entirely. If formatting changes meaning (e.g. JSX whitespace
   that becomes a visible space), report it; otherwise skip.

---

## 7. Padding the report

❌ "Great work overall! I really like the structure you chose. One small
    thing…"
✅ Lead with findings. No praise, no closing pep-talk.

---

## 8. Speculative findings

❌ "This might be slow under load."
✅ "Hot loop in `foo:42` performs an O(n) lookup inside an O(n) iteration —
    O(n²) for n>~1k. Recommend a Map."

If the speculative path is genuinely a risk but unverifiable from the diff,
file it under Suggestions and explicitly mark uncertainty.

---

## 9. Lowering severity to seem nicer

❌ Marking a missing auth check as "Minor" to soften the report.
✅ Apply the rubric. Auth = Critical. Always.

---

## 10. APPROVED with open Majors

❌ "APPROVED, but please address these major issues before merging."
✅ `NEEDS_CHANGES` whenever any Critical, Major, or failed AC exists.

---

## 11. Closing with "want me to apply these fixes?"

❌ Offering to make the changes from inside the reviewer skill.
✅ The escalation handoff sentence from `SKILL.md § Escalation`:
   *"Out of scope for the reviewer skill — invoke `/quick-task` for a single
   small fix, or `/story-creator` if multiple changes are needed."*

---

## 12. Re-reviewing without a new diff

❌ Re-running review on the same unchanged tree to "double-check."
✅ One pass per diff. Re-review only after the diff changes.
