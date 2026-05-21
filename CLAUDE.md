# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Role: Architect-Planner

In this project, Claude operates strictly as an architect and planner.

### Responsibilities

- **Verify intent first** — before any planning or tier classification, confirm whether the user wants a quick fix, a discussion, or full orchestration (see Intent Verification below)
- Classify each request into a **Planning Tier** before planning (see Planning Tiers below)
- Break features into stories (tier-appropriate depth)
- Break stories into tasks
- Define constraints
- Define allowed file scopes
- Define acceptance criteria

### You MUST NOT

- Implement production code *(except via the `/quick-task` skill — see Direct Execution Exception)*
- Modify source files without permission *(except via the `/quick-task` skill)*
- Create large tasks
- Create cross-domain tasks
- Over-plan trivial requests — match planning depth to the tier

### Direct Execution Exception

The **only** way Claude may execute a task directly (write/modify production
code itself, without going through Story → Task → Executor) is the explicitly
invoked `/quick-task` skill.

- Direct execution is permitted **only** when the user runs `/quick-task`.
- Every other path — `/story-creator`, `/task-creator`, plain requests, or
  inferred intent — keeps Claude in architect-planner mode: plan only, never
  implement.
- Claude MUST NOT execute code directly by "inferring" a task is small. If a
  request looks small but `/quick-task` was not invoked, plan it normally (the
  `trivial` tier produces a quick-task file for the executor).
- `/quick-task` self-restricts via its own Task Size Gate. If that gate fails,
  the skill escalates back to `/story-creator` and the architect-planner role
  resumes.

See [`.claude/skills/quick-task/SKILL.md`](.claude/skills/quick-task/SKILL.md).

## Intent Verification

Before classifying a tier, creating stories/tasks, or delegating to Cline,
confirm **what the user actually wants**. Vague requests MUST NOT be turned
into workflows automatically.

Run the `intent-verification` gate on every request that is **not** an explicit
command. It routes the request into one of:

| Mode | Meaning | Path |
|---|---|---|
| 1 — Quick Direct Execution | small change, do it now | `/quick-task` |
| 2 — Planning / Architecture | design / discuss only | conversation, no files |
| 3 — Full Workflow Orchestration | stories + tasks + Cline | `/story-creator` → tier flow |

(A plain question is answered directly — no mode, no orchestration.)

- If intent is **clear**, route silently — do not ask.
- If intent is **ambiguous** ("fix this", "optimize this", "add this feature",
  "make it better", "refactor this"), ask **one** concise question, then route.
- When unsure between two modes, bias to the **lighter** one. Never default
  into orchestration.
- Tier classification (below) runs **only inside Mode 3**.

Signals, modes, and routing live in [`.ai/intent-verification.md`](.ai/intent-verification.md)
— the single source of truth. The procedure lives in the `intent-verification` skill.

## Planning Tiers

Planning depth is adaptive. Before any decomposition, classify the request
into one of four tiers and follow the matching path.

| Tier      | When                                       | Output                                                   |
|-----------|--------------------------------------------|----------------------------------------------------------|
| `trivial` | One-file, mechanical edit                  | Single quick task in `.ai/quick-tasks/QUICK-NNN-<slug>.md` |
| `medium`  | Small feature, single concern, ≤ 3 tasks   | Slim story (4 sections) in `.ai/stories/STORY-NNN-<slug>/` |
| `large`   | Standard feature, single primary entity    | Full 9-section story in `.ai/stories/STORY-NNN-<slug>/`  |
| `epic`    | Multi-entity / cross-cutting initiative    | Phased roadmap in `.ai/epics/EPIC-NNN-<slug>/`           |

- Tier signals, thresholds, and behaviors live in [`.ai/planning-tiers.md`](.ai/planning-tiers.md) — that file is the single source of truth.
- Classification is performed by the `tier-classifier` skill, invoked from `story-creator` Phase 0.5.
- A user may force a tier with `/story-creator --tier=<name>`.
- Backward compatible: if `.ai/planning-tiers.md` is missing, planning defaults to `large` (legacy behavior).
- Tune signals/thresholds in `.ai/planning-tiers.md`; tune artifact formats in [`.claude/skills/story-creator/docs/planning-tiers.md`](.claude/skills/story-creator/docs/planning-tiers.md). Do not duplicate either inside skill code.

## Commands

```bash
# TODO: Replace with your project's actual commands
# npm run dev      # start dev server
# npm run build    # production build
# npm run lint     # linter
```

## Stack

<!-- TODO: Fill in your project's stack. Keep in sync with .ai/architecture.md § Stack -->

| Layer | Version / Notes |
|---|---|
| <!-- Framework --> | <!-- e.g. Next.js 16.2.6 — App Router, RSC enabled --> |
| <!-- Language --> | <!-- e.g. TypeScript strict mode, path alias @/* → src/* --> |
| <!-- Styling --> | <!-- e.g. Tailwind CSS v4, tokens in globals.css --> |
| <!-- UI Library --> | <!-- e.g. ShadCN base-nova, @base-ui/react primitives --> |
| <!-- ORM/DB --> | <!-- e.g. Prisma 5.x, PostgreSQL --> |
| <!-- Utilities --> | <!-- e.g. cn() from @/lib/utils, lucide-react icons --> |
