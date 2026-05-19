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
│   └── stories/                          # Stories and tasks live here (populated at runtime)
├── .claude/
│   └── skills/
│       ├── story-creator/                # Skill: requirement → bounded stories
│       └── task-creator/                 # Skill: story → file-scoped tasks
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

4. **Start planning** — in Claude Code:
   ```
   /story-creator "your feature description"
   ```

5. **Start executing** — in Cline, for each task:
   ```
   Run STORY-NNN-<slug> TASK-001
   ```

---

## Workflow at a glance

```
User requirement
      │
      ▼
/story-creator "feature"          ← Claude Code decomposes into stories
      │  writes .ai/stories/STORY-NNN-<slug>/story.md
      │
      ▼
/task-creator STORY-NNN-<slug>    ← Claude Code decomposes story into tasks
      │  writes .ai/stories/STORY-NNN-<slug>/tasks/TASK-NNN-<slug>.md
      │
      ▼
Cline executes TASK-001           ← Cline reads task, implements, updates context
      │  modifies source files
      │  appends to context.md
      │
      ▼
Human reviews → triggers TASK-002 ← Repeat until story complete
```

---

## Key files to customize per project

| File | What to fill in |
|---|---|
| `.ai/architecture.md` | Stack, stack rules, entities, path conventions, commands |
| `CLAUDE.md` | Stack summary, build commands |

All other files (skills, clinerules) are generic and work unchanged across projects.

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
