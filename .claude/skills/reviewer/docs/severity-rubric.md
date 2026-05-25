# Severity Rubric — Decision Examples

The reviewer uses four severity levels. They are not personal opinion — they
are decision categories. Use these examples to calibrate.

---

## Critical

Ship-blockers. Data loss, security, broken core flow, broken build, violated
acceptance criterion.

- A migration drops a column that production code still reads.
- Auth check is `if (user)` instead of `if (user.isAdmin)` on an admin route.
- API key logged in plaintext to stdout.
- SQL built via string concatenation from request body.
- Acceptance criterion AC-2 says "endpoint returns 401 on missing token";
  diff returns 200.
- New code path throws on the happy path of the main user flow.

---

## Major

Significant defect or regression risk. Not catastrophic, but should not merge.

- Missing `await` on a write whose result the caller depends on.
- Public function changed signature without updating any of its 4 call sites.
- Required test for new branch logic is missing.
- New module introduces a second source of truth for an existing concept.
- Effect dependency array omits a value the effect reads (stale closure).
- Error swallowed by an empty `catch {}` on a code path that can fail
  meaningfully.

---

## Minor

Real issue, narrow impact.

- Typo in a log message.
- Edge case unlikely to occur in practice (e.g. `null` from a function that
  in practice never returns null) but not guarded.
- Unused import.
- Function name no longer matches what it does after the change.
- Comment now lies about the code below it.

---

## Suggestions

Improvements, not defects. The author may decline.

- Could extract a helper used twice.
- Alternative API would read better.
- Could memoize this computation if profile shows it matters.
- Naming nit that does not change meaning.

---

## Tie-Breaker Rules

- Uncertain between Critical and Major → pick Major and note the
  uncertainty.
- Uncertain between Major and Minor → pick Minor.
- Uncertain between Minor and Suggestion → pick Suggestion.
- If a finding affects production data or auth → never lower than Major.
- Style-only nits → drop entirely, do not emit as Suggestion.

## Final Status

- Any Critical → `NEEDS_CHANGES`.
- Any Major → `NEEDS_CHANGES`.
- Any failed acceptance criterion → `NEEDS_CHANGES`.
- Otherwise → `APPROVED`.
