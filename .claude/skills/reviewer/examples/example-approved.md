# Example Review — APPROVED (no significant issues)

Shows the shape of the report when nothing meaningful is found. The severity
sections collapse to a single line; the Acceptance Criteria section and Final
Status are still emitted.

---

# Review Summary

**Scope:** branch `chore/rename-userId-to-accountId` vs `main` (QUICK-017)
**Files reviewed:** 4
**Diff size:** ~38 changed

Mechanical rename of `userId` → `accountId` across the billing module. All
call sites updated, types align, no behavioral change.

No significant issues found in reviewed changes.

## Acceptance Criteria Check
- [x] AC-1: All references to `userId` in `src/billing/**` renamed to `accountId` — Passed.
- [x] AC-2: No remaining references to `userId` in renamed files — Passed (grep clean).
- [x] AC-3: Existing tests still compile and assert the same behavior — Not verifiable from diff. Would be confirmed by running `npm test`.

## Final Status
APPROVED
