---
name: tier-classifier
description: >
  Classify an incoming planning request into a Planning Tier
  (trivial / medium / large / epic) using the signals and thresholds defined in
  .ai/planning-tiers.md. Returns a tier plus a one-line rationale. Used by
  story-creator at Phase 0.5 and may be invoked directly via /tier-classify.
version: 1.0.0
---

# tier-classifier — Planning Tier Classifier

Reads `.ai/planning-tiers.md` and applies its signal weights + thresholds to a
planning request. Emits a tier label and a short rationale. Does **not** write
any files.

**This skill is a pure function: input → tier. It does not plan or implement.**

---

## Invocation Patterns

```
/tier-classify "Add a search bar to the notes list"
/tier-classify --prd path/to/requirements.md
```

Also invoked internally by `story-creator` Phase 0.5.

---

## Workflow

### Phase 0 — Load Config

1. Read `.ai/planning-tiers.md`. Stop if missing and tell the user to create it.
2. Read `.ai/architecture.md § Path Conventions` and `§ Domain` to ground file
   and module counts in the project's actual structure.

### Phase 1 — Extract Signals

For each signal in `planning-tiers.md § Complexity Signals`, derive a value
from the request:

| Signal                            | How to derive                                                  |
|-----------------------------------|----------------------------------------------------------------|
| `estimated_files`                 | Map the requested concerns onto path conventions, count files. |
| `scope_keywords`                  | Substring match against the request text (case-insensitive).   |
| `layer_span`                      | Count layers (data, service, API, UI) implied by the request.  |
| `modules_involved`                | Count distinct top-level modules in `architecture.md`.         |
| `migration_or_infra`              | yes if request mentions DB migration, infra, CI, env vars.     |
| `ui_plus_backend`                 | yes if request introduces both new UI and new backend code.    |
| `test_impact`                     | none / scoped / broad — based on test surface implied.         |
| `architectural_decision_required` | yes if a new pattern/decision is needed (no prior precedent).  |

Be honest. If a signal is ambiguous, lean **up** (more complexity → more planning).

### Phase 2 — Score

Sum the per-signal weights. Apply **Tie-breakers** in the order listed in
`planning-tiers.md § Tier Thresholds`.

### Phase 3 — Map to Tier

Apply the threshold ranges from `planning-tiers.md § Tier Thresholds`. If the
user passed `--tier=<name>`, that overrides the computed tier.

### Phase 4 — Emit Result

Output a single fenced block (parseable by `story-creator`):

```
TIER: <trivial|medium|large|epic>
SCORE: <integer>
SIGNALS:
  estimated_files: <value> (+w)
  scope_keywords: <matched terms> (+w)
  layer_span: <value> (+w)
  modules_involved: <value> (+w)
  migration_or_infra: <yes|no> (+w)
  ui_plus_backend: <yes|no> (+w)
  test_impact: <none|scoped|broad> (+w)
  architectural_decision_required: <yes|no> (+w)
TIE_BREAKERS_APPLIED: <list or none>
RATIONALE: <one sentence>
```

Stop. The caller decides what to do next.

---

## Examples

**Trivial**

```
Request: "Fix typo in the homepage hero title."
TIER: trivial
SCORE: -1
SIGNALS:
  estimated_files: 1 (+0)
  scope_keywords: typo (-2)
  layer_span: 1 (+0)
  ... (zeros)
TIE_BREAKERS_APPLIED: ["1-file rename/typo → force trivial"]
RATIONALE: Single-file copy edit, no layers, no logic.
```

**Medium**

```
Request: "Add an `archived` boolean filter to the notes list query."
TIER: medium
SCORE: 4
SIGNALS:
  estimated_files: 2-3 (+1)
  scope_keywords: add field (+1)
  layer_span: 2 (+2)
  ... (zeros)
RATIONALE: Two-layer change, one entity, ~3 files.
```

**Large**

```
Request: "Add a notes search feature (API + UI)."
TIER: large
SCORE: 9
SIGNALS:
  estimated_files: 4-7 (+3)
  scope_keywords: feature, endpoint, component (+3)
  layer_span: 3 (+4)
  ui_plus_backend: yes (+3)
  ... (zeros, after dedupe)
RATIONALE: Full-stack feature on a single entity.
```

**Epic**

```
Request: "Migrate users from MySQL to Postgres, update auth service and admin UI."
TIER: epic
SCORE: 21
SIGNALS:
  estimated_files: 16+ (+10)
  scope_keywords: migration, multi-service (+6)
  layer_span: 4 (+6)
  modules_involved: 3+ (+5)
  migration_or_infra: yes (+4)
  ui_plus_backend: yes (+3)
  architectural_decision_required: yes (+5)
TIE_BREAKERS_APPLIED: ["migration_or_infra + modules>=2 → force epic"]
RATIONALE: Cross-cutting migration spanning multiple modules.
```

---

## Constraints

- This skill MUST NOT create or modify files (no `.ai/` writes, no source writes).
- This skill MUST NOT invoke `story-creator` or `task-creator`.
- If `.ai/planning-tiers.md` is missing, stop and report — do not guess defaults.
