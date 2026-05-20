# Planning Tiers — Branch Specifications

Concrete artifact formats and phase paths for each tier defined in
`.ai/planning-tiers.md`. The classifier picks the tier; this file says
**what to produce** once a tier is picked.

If you change a tier's signals, thresholds, or high-level behavior, edit
`.ai/planning-tiers.md`. If you change a tier's **artifact format**, edit this
file.

---

## How tier selection works (summary)

1. `story-creator` Phase 0.5 invokes `tier-classifier`.
2. `tier-classifier` reads `.ai/planning-tiers.md`, scores the request, returns
   one of `trivial | medium | large | epic` + rationale.
3. `story-creator` branches to the matching section in this document.
4. The user may bypass classification with `--tier=<name>`.

---

## Trivial — Quick Task path

**When**: classifier returns `trivial`.

**Artifact**: one file at `.ai/quick-tasks/QUICK-NNN-<slug>.md`. No story dir,
no `context.md`, no logs. NNN is zero-padded, incremented from the highest
existing `QUICK-NNN` in `.ai/quick-tasks/`.

**Phases**: skip Phases 1–6 of the full workflow. Do only:

1. Generate the quick-task file from the template below.
2. Print the one-line summary.
3. Stop.

**Template**:

```markdown
# QUICK-NNN-<slug> — <Task Title>

**Tier:** trivial
**Tier Rationale:** <one line from classifier>
**Created:** <ISO date>

## Objective

<one to three sentences — exact change to make>

## Allowed Files

- `<exact/path/to/file>` `[modify]`

## Acceptance Criteria

- <one or two mechanically verifiable criteria>

## Context Update

After completion, append to `.ai/quick-tasks/quick-log.md`:

```
### QUICK-NNN [<ISO date> <HH:MM>] — <Title>
- Status: COMPLETED
- Files: <list>
```
```

**Executor behavior**: the existing `executor` workflow already handles
file-scoped tasks. Quick tasks reuse its Phase 2/3/5/6; Phase 1 just loads the
quick-task file instead of `story.md + context.md + task.md`.

---

## Medium — Slim Story path

**When**: classifier returns `medium`.

**Artifact**: one story at `.ai/stories/STORY-NNN-<slug>/` using the **slim**
template. Only 4 required sections — the other 5 from `story-format.md` are
omitted to cut tokens.

**Phases**:

1. **Slim Phase 1 (Feature Analysis lite)**: 3 bullets, not the full DOMAIN /
   DELIVERY block.
2. **Skip Phase 2 (Story Boundary)** — medium tier is single-story by definition.
3. **Phase 3 sizing**: enforce `max_tasks: 3` instead of 7. If exceeded → escalate
   tier to `large` and restart from Phase 0.5.
4. **Phase 4 (Story File)**: write the slim story below.
5. **Phase 5 (Task Orchestration)**: call `/task-creator STORY-NNN-<slug>` as
   usual. task-creator does not need to know about tiers.
6. **Phase 6 (Summary)**: print short summary.

**Slim story template** (required sections only):

```markdown
# STORY-NNN-<slug> — <Story Title>

**Tier:** medium
**Tier Rationale:** <one line>

## Goal

<user-observable outcome>

## Acceptance Criteria

- <≥ 3 mechanically verifiable criteria>

## Affected Areas

- `<exact/path>` — <one-line role>

## Constraints

<verbatim copy from .ai/architecture.md § Stack Rules>
```

`context.md` is initialized with a slim header (just `## Status` and
`## Completed Tasks`).

---

## Large — Full Story path (default)

**When**: classifier returns `large`, **or** no tier system is in play.

**Artifact**: one full 9-section story at `.ai/stories/STORY-NNN-<slug>/`,
exactly as defined in `docs/story-format.md`.

**Phases**: run Phases 1–6 of the main `SKILL.md` workflow unchanged. This is
the legacy behavior — no tier-specific deviation.

Add only this line to the story header:

```markdown
**Tier:** large
**Tier Rationale:** <one line>
```

---

## Epic — Phased path

**When**: classifier returns `epic`.

**Artifact**: `.ai/epics/EPIC-NNN-<slug>/` containing:

```
EPIC-NNN-<slug>/
├── epic.md                 # phased roadmap + dependency map
└── stories/                # populated phase by phase
    ├── STORY-NNN-<slug>/   # generated for Phase 1 only on this run
    └── ...                 # later phases: regenerate later on demand
```

**Phases**:

1. **Phase 1 (Feature Analysis)** — full version.
2. **Phase 1.5 (Phase Planning)** — split the epic into 2–N **phases**. Each
   phase is an ordered group of stories that can be completed before the next
   phase begins. Record phases in `epic.md`.
3. **Phase 2 (Story Boundary)** — apply to each phase independently.
4. **Per-phase story generation**: generate full stories **only for Phase 1**.
   Do not generate stories for later phases until the user invokes
   `/story-creator EPIC-NNN-<slug> --phase=2` (or whichever phase).
5. **Phase 5 (Task Orchestration)**: invoke `task-creator` for each generated
   Phase 1 story.
6. **Phase 6 (Summary)**: print the phased roadmap.

**epic.md template**:

```markdown
# EPIC-NNN-<slug> — <Epic Title>

**Tier:** epic
**Tier Rationale:** <one line>
**Created:** <ISO date>

## Goal

<one paragraph — what the epic delivers, why>

## Phases

### Phase 1 — <Phase Name>

Goal: <one line>
Stories:
  - STORY-NNN-<slug> — <Title>
  - STORY-NNN-<slug> — <Title>
Exit criteria: <observable signal that phase is done>

### Phase 2 — <Phase Name>  [NOT YET GENERATED]

Goal: <one line>
Planned stories (titles only):
  - <Title>
  - <Title>
Depends on: Phase 1 complete.

### Phase 3 — ... [NOT YET GENERATED]

## Dependency Map

```
Phase 1 ──▶ Phase 2 ──▶ Phase 3
   │            │
   └── STORY-001
   └── STORY-002
```

## Active Phase

Phase 1 (stories generated, tasks generated, ready to execute).
```

**Invocation for later phases**:

```
/story-creator EPIC-NNN-<slug> --phase=2
```

This generates stories + tasks for Phase 2 only. The classifier is bypassed
(the epic decision is already made).

---

## Customizing thresholds

To make the system route smaller tasks into deeper tiers (or vice versa):

1. Open `.ai/planning-tiers.md`.
2. Tune `signals.*.weights` to reweight specific signals.
3. Tune `thresholds` to shift tier boundaries.
4. To add a new signal, append it under `signals:` and reference it in
   `tier-classifier/SKILL.md § Phase 1 — Extract Signals`.

No skill code changes are needed for routine threshold tuning.

---

## Backward compatibility

- Projects that have not created `.ai/planning-tiers.md` continue to work:
  `story-creator` falls back to the `large` (full story) path with a warning
  the first time it sees a missing config.
- All existing stories under `.ai/stories/` remain valid — they are
  implicitly `large` tier.
- `task-creator` is unchanged. Tiers affect only what story-creator emits.
