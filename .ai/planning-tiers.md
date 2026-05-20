# Planning Tiers

Declarative configuration for adaptive planning depth.

The planning system reads this file to decide **how much planning** an incoming
task warrants. Lower-complexity work skips story decomposition and goes straight
to a single bounded task; higher-complexity work gets the full story/task/epic
treatment.

This file is the **single source of truth** for tier behavior. Skills and
workflows read it — do not duplicate thresholds elsewhere.

---

## Tiers

| Tier      | Output artifact                                       | Story count | Tasks per story | Architecture analysis | Use when                                |
|-----------|-------------------------------------------------------|-------------|-----------------|-----------------------|-----------------------------------------|
| `trivial` | Single quick-task in `.ai/quick-tasks/`               | 0           | n/a (1 task)    | none                  | One-file, mechanical edit               |
| `medium`  | One slim story in `.ai/stories/`                      | 1           | ≤ 3             | inline (≤ 3 bullets)  | Small feature, single concern           |
| `large`   | One full story in `.ai/stories/` (current behavior)   | 1           | ≤ 7             | full                  | Standard feature, single primary entity |
| `epic`    | Epic dir `.ai/epics/EPIC-NNN-<slug>/` + N stories     | 2+          | ≤ 7 each        | phased + deps map     | Multi-entity / cross-cutting initiative |

---

## Complexity Signals

Each signal contributes a score. The classifier sums them, then maps the score
to a tier via the thresholds below. **All weights and thresholds are tunable
here without touching skill code.**

```yaml
signals:
  estimated_files:
    description: "Estimated number of source files touched."
    weights:
      "1":         0
      "2-3":       1
      "4-7":       3
      "8-15":      6
      "16+":       10

  scope_keywords:
    description: "Keywords in the request that imply broad scope."
    weights:
      # Trivial-leaning
      "rename|typo|comment|copy|wording": -2
      "bump version|update dep": -1
      # Medium-leaning
      "add field|add prop|small fix|tweak": 1
      # Large-leaning
      "feature|endpoint|component|page|hook|service": 3
      # Epic-leaning
      "migration|infra|database schema|cross-cutting|platform|multi-(service|module)": 6
      "refactor (system|architecture)|rewrite|overhaul": 6

  layer_span:
    description: "How many layers the change touches (data → service → API → UI)."
    weights:
      "1": 0
      "2": 2
      "3": 4
      "4": 6

  modules_involved:
    description: "Number of distinct top-level modules/packages affected."
    weights:
      "1": 0
      "2": 2
      "3+": 5

  migration_or_infra:
    description: "Does the work touch DB migrations, infra config, or CI?"
    weights:
      "no":  0
      "yes": 4

  ui_plus_backend:
    description: "Does the work combine new UI and new backend in one request?"
    weights:
      "no":  0
      "yes": 3

  test_impact:
    description: "Does the change require new/modified test suites beyond unit-of-change?"
    weights:
      "none":     0
      "scoped":   1
      "broad":    3

  architectural_decision_required:
    description: "Does the planner need to make a new architectural decision?"
    weights:
      "no":  0
      "yes": 5
```

---

## Tier Thresholds

```yaml
thresholds:
  trivial: "<= 1"     # score <= 1
  medium:  "2..5"     # 2 to 5 inclusive
  large:   "6..12"    # 6 to 12 inclusive
  epic:    ">= 13"    # 13 or more
```

**Tie-breakers** (applied in order, override score-based tier):

1. If `migration_or_infra=yes` AND `modules_involved>=2` → force `epic`.
2. If `architectural_decision_required=yes` AND tier was `trivial`/`medium` → bump to `large`.
3. If `estimated_files >= 16` → force `epic`.
4. If `estimated_files == 1` AND `scope_keywords` is rename/typo/comment → force `trivial`.

---

## Tier Behaviors

What each tier produces and which planning phases run.

### `trivial`

```yaml
output:
  path: ".ai/quick-tasks/QUICK-NNN-<slug>.md"
  format: "single-task template (see story-creator/docs/planning-tiers.md § Quick Task Format)"
phases_run:    [classify, write_quick_task]
phases_skip:   [feature_analysis, story_boundary, sizing_check, task_orchestration]
context_file:  none
log_file:      none
notes: "Executor runs the quick task directly. No context.md, no story dir."
```

### `medium`

```yaml
output:
  path: ".ai/stories/STORY-NNN-<slug>/"
  story_sections_required: [Goal, Acceptance Criteria, Affected Areas, Constraints]
  story_sections_optional: [Business Context, Technical Decisions, Dependencies, Execution Strategy, Risks]
  max_tasks: 3
phases_run:    [classify, slim_feature_analysis, slim_story, task_orchestration]
phases_skip:   [story_boundary_split, full_architecture_analysis]
context_file:  ".ai/stories/STORY-NNN-<slug>/context.md (initialized, slim header)"
notes: "Skip the 9-section story template. Use a 4-section slim story."
```

### `large` (default — current behavior)

```yaml
output:
  path: ".ai/stories/STORY-NNN-<slug>/"
  story_sections_required: "all 9 sections (see story-creator/docs/story-format.md)"
  max_tasks: 7
phases_run:    [classify, feature_analysis, story_boundary, sizing_check, story_file, task_orchestration]
phases_skip:   []
context_file:  ".ai/stories/STORY-NNN-<slug>/context.md"
notes: "Default. Full story-creator workflow as previously documented."
```

### `epic`

```yaml
output:
  path: ".ai/epics/EPIC-NNN-<slug>/"
  artifacts:
    - "epic.md (phased roadmap, dependency map, story index)"
    - "stories/STORY-NNN-<slug>/ (one or more full stories, generated phase by phase)"
phases_run:    [classify, feature_analysis, phase_planning, story_boundary, per_phase_story_gen, dependency_map, task_orchestration]
phases_skip:   []
phase_strategy: "Plan phases first. Generate stories only for Phase 1. Stop and report. Subsequent phases generated on demand."
notes: "Avoids planning beyond what the team will act on this iteration."
```

---

## Override

A user may force a tier explicitly:

```
/story-creator --tier=medium "Add search to notes"
/story-creator --tier=epic --prd path/to/big-prd.md
```

When `--tier` is set, the classifier is bypassed and the chosen tier behavior
applies directly. The forced tier is recorded in the story/epic metadata.

---

## Extending

To add or tune a tier:

1. Add/modify entries in **Tiers**, **Complexity Signals**, **Tier Thresholds**,
   and **Tier Behaviors** sections.
2. If a new tier introduces a new artifact layout, document it in
   `.claude/skills/story-creator/docs/planning-tiers.md`.
3. Do not duplicate config inside skill files — they read this file.
