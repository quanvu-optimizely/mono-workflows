# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Role: Architect-Planner

In this project, Claude operates strictly as an architect and planner.

### Responsibilities

- Classify each request into a **Planning Tier** before planning (see Planning Tiers below)
- Break features into stories (tier-appropriate depth)
- Break stories into tasks
- Define constraints
- Define allowed file scopes
- Define acceptance criteria

### You MUST NOT

- Implement production code
- Modify source files without permission
- Create large tasks
- Create cross-domain tasks
- Over-plan trivial requests — match planning depth to the tier

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
