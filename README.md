# mono-workflows

A reusable AI workflow library for the two-agent (Claude Code + Cline) planning and execution model.

Copy this library into any project to adopt a structured, auditable development workflow where Claude Code plans and Cline executes — with strict separation of concerns between the two agents.

---

## What's included

```
mono-workflows/
├── CLAUDE.md                              # Claude Code's architect-planner role (template)
├── .ai/
│   ├── architecture.md                   # Project knowledge base (fill in per project)
│   ├── intent-verification.md            # Intent routing modes, signals, and rules
│   ├── planning-tiers.md                 # Adaptive planning depth config (signals, thresholds, behaviors)
│   ├── stories/                          # Stories and tasks live here (populated at runtime)
│   ├── quick-tasks/                      # Tier=trivial single-task artifacts
│   └── epics/                            # Tier=epic phased roadmaps + child stories
├── .claude/
│   └── skills/
│       ├── intent-verification/          # Skill: route request → mode before any planning
│       ├── tier-classifier/              # Skill: request → planning tier
│       ├── story-creator/                # Skill: requirement → bounded stories (tier-aware)
│       ├── task-creator/                 # Skill: story → file-scoped tasks
│       └── quick-task/                   # Skill: direct execution bypass for trivial changes
└── .clinerules/
    ├── execution-agent.md                # Cline's role and constraints
    └── workflows/
        └── executor.md                  # Cline's 6-phase execution workflow
```

---

## Two-agent model

| Agent | Tool | Responsibility | Writes to |
|---|---|---|---|
| **Architect-Planner** | Claude Code | Stories, tasks, architecture decisions | `.ai/` only |
| **Executor** | Cline | Source code, context updates | Files listed in each task's `Allowed Files` |

**Invariants:**
- Claude Code NEVER writes production source code.
- Cline NEVER makes architecture decisions — it records them in `context.md`.
- Tasks are executed one at a time. Cline stops after each task and waits for human trigger.

---

## Adopting this workflow in a new project

1. **Copy** the entire `mono-workflows/` directory into your project root:
   ```bash
   cp -r mono-workflows/. your-project/
   ```

2. **Fill in** `.ai/architecture.md` with your project's stack, entities, path conventions, and commands. This file is read by both skills in Phase 0 — it is the single source of truth.

3. **Update** `CLAUDE.md` with your project's stack summary and build commands.

4. **Start planning** — in Claude Code, just describe what you need:
   ```
   Add user authentication via OAuth
   ```

   Claude Code will first run the **intent verification** gate to confirm
   whether you want a quick fix, a discussion, or full orchestration — then
   route automatically. For explicit control:

   | Command | What it does |
   |---|---|
   | `/quick-task "..."` | Execute a small change directly (no orchestration) |
   | `/story-creator "..."` | Full story + task decomposition |
   | `/story-creator --tier=<name> "..."` | Force a planning tier |

   `story-creator` consults `.ai/planning-tiers.md` to choose a planning depth
   automatically — `trivial`, `medium`, `large`, or `epic`. See
   [`.claude/skills/story-creator/docs/planning-tiers.md`](.claude/skills/story-creator/docs/planning-tiers.md)
   for the artifact format each tier produces.

5. **Start executing** — in Cline, for each task:
   ```
   Run STORY-NNN-<slug> TASK-001
   ```

---

## Workflow at a glance

```
User request
      │
      ▼
Intent Verification               ← What does the user actually want?
      │
      ├── Question / discussion   → Answer directly (no files written)
      │
      ├── Mode 1: small change    → /quick-task  ← direct execution, no orchestration
      │   writes .ai/quick-tasks/QUICK-NNN-<slug>.md
      │
      └── Mode 3: full workflow   ↓
            │
            ▼
      /story-creator "feature"    ← classify tier, decompose into stories
            │  writes .ai/stories/STORY-NNN-<slug>/story.md
            │
            ▼
      /task-creator STORY-NNN     ← decompose story into file-scoped tasks
            │  writes .ai/stories/STORY-NNN-<slug>/tasks/TASK-NNN-<slug>.md
            │
            ▼
      Cline executes TASK-001     ← reads task, implements, updates context.md
            │
            ▼
      Human reviews → TASK-002   ← repeat until story complete
```

---

## Key files to customize per project

| File | What to fill in |
|---|---|
| `.ai/architecture.md` | Stack, stack rules, entities, path conventions, commands |
| `CLAUDE.md` | Stack summary, build commands |
| `.ai/intent-verification.md` | Tune ambiguity signals and routing bias (optional) |
| `.ai/planning-tiers.md` | Tune tier thresholds and tier signals (optional) |

The skill files and `.clinerules/` are generic and work unchanged across projects.

---

## Directory conventions

Stories live at:
```
.ai/stories/STORY-NNN-<slug>/
├── story.md           # Story definition (9 sections)
├── context.md         # Append-only execution context
├── tasks/
│   ├── TASK-001-<slug>.md
│   └── TASK-002-<slug>.md
└── logs/
    └── execution-log.md
```

`NNN` is zero-padded (001, 002, …). Slugs are 3–5 word kebab-case summaries.
