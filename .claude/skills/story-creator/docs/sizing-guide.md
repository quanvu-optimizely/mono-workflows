# Story Sizing Guide

Use this guide to determine whether a story is correctly sized and how to split
it when it fails a gate.

---

## Five Gates

Run every gate before finalizing any story. Fail = the story must be split.

### Gate 1 — Task Count

```
LIMIT : ≤ 7 tasks after full decomposition
METHOD: Mentally apply task-creator decomposition to the story.
        If you count > 7 tasks, split the story.
```

7 tasks is the upper bound. 4–6 is the target range.
A story with 3 tasks is fine. A story with 2 tasks may need to merge with
an adjacent story (see Merge Conditions below).

### Gate 2 — Primary Entity Count

```
LIMIT : = 1 primary entity
METHOD: Count how many entities (from .ai/architecture.md § Domain § Entities)
        own data layer changes in this story.
        If > 1, split by entity.
```

A story MAY reference a second entity's types (e.g., a reference ID from another
entity) without failing this gate — as long as the second entity's data layer
is not modified.

### Gate 3 — Layer Span Risk

```
LIMIT : Full stack (data → UI) for one entity is allowed per story.
        Cross-entity UI is NOT allowed.
METHOD: Check if the UI layer in this story renders data from more than
        one primary entity's API. If yes, split by entity.
```

Example: A "`<Entity1>` List + `<Entity2>` Panel" story spans two entities in
the UI. Split into "`<Entity1>` List View" and "`<Entity2>` Panel".

### Gate 4 — Executor Decision Risk

```
LIMIT : No decision that requires real-time architecture judgment.
METHOD: Review Technical Decisions section. If any item uses:
        "choose appropriate", "decide based on context", "implement as needed" →
        FAIL. Make the decision now, in the story file.
```

If you genuinely cannot make a decision (missing information, pending spike),
create a Spike task as TASK-001 and block all subsequent tasks on it.

### Gate 5 — Dependency Depth

```
LIMIT : ≤ 2 upstream stories
METHOD: Count Stories listed in the Dependencies section.
        If > 2, the story is too late in the dependency chain — split its
        layers into earlier, smaller stories.
```

---

## Six Story Classifications

| Class | Task Count | Layer Span | Example |
|---|---|---|---|
| **Data Foundation** | 3–5 | schema + types + repo + service | `<Entity>` Data Layer Foundation |
| **API Layer** | 2–4 | service → API routes | `<Entity>` CRUD API |
| **UI View** | 3–5 | hook + components + page | `<Entity>` List View |
| **Feature Slice** | 4–7 | data + API + UI for one entity | `<Entity>` Creation Flow |
| **Utility** | 1–3 | single shared module | Input Validator |
| **Integration** | 2–4 | wires two existing layers | `<Entity>`-`<Entity2>` Association |

A "Feature Slice" is the largest allowed class. If a feature slice exceeds 7
tasks, split into Data Foundation + API Layer + UI View.

---

## Five Split Patterns

### Split Pattern 1 — By Layer

Use when a full-stack story exceeds 7 tasks.

```
BEFORE: <Entity> Full Feature (schema + types + repo + service + API + hook + list + detail + form + page)
        → 10 tasks

AFTER:
  STORY-001  <Entity> Data Layer    (schema + migration + types + repo + service)     → 5 tasks
  STORY-002  <Entity> API Layer     (POST route + GET route + GET-by-id route)         → 3 tasks
  STORY-003  <Entity> List View     (GET hook + <Entity>Card + <Entity>List + list page)  → 4 tasks
  STORY-004  <Entity> Detail View   (GET-by-id hook + <Entity>Detail + detail page)   → 3 tasks
```

### Split Pattern 2 — By Entity

Use when a story mixes two primary entities.

```
BEFORE: <Entity1> + <Entity2> Data Layer (both models + both repos + both services)
        → 10 tasks

AFTER:
  STORY-001  <Entity1> Data Layer Foundation    → 5 tasks
  STORY-003  <Entity2> Data Layer               → 4 tasks
```

### Split Pattern 3 — By HTTP Method Group

Use when an API story generates > 4 tasks because it covers all CRUD verbs.

```
BEFORE: <Entity> CRUD API (POST + GET + GET-by-id + PUT + DELETE + validator + middleware)
        → 7 tasks (borderline — still acceptable)

AFTER (if it grows):
  STORY-002A  <Entity> Read API    (GET + GET-by-id + hook)           → 3 tasks
  STORY-002B  <Entity> Write API   (POST + PUT + DELETE + validator)   → 4 tasks
```

### Split Pattern 4 — By Component

Use when a UI story has too many components for a single story.

```
BEFORE: <Entity> Editor UI (<Form> + RichTextEditor + TagPicker + MediaWidget + page)
        → 6 components → 7+ tasks

AFTER:
  STORY-003  <Entity> Editor Core   (<Form> + editor page)             → 4 tasks
  STORY-007  <Entity> Editor Extras (TagPicker + MediaWidget)          → 4 tasks
```

### Split Pattern 5 — Spike Extraction

Use when a story contains unresolved technical unknowns.

```
BEFORE: Auth Integration (auth middleware — strategy TBD)
        → cannot task until auth strategy is decided

AFTER:
  STORY-009  Auth Strategy Spike   (research + ADR document)   → 1–2 tasks
  STORY-010  Auth Middleware        (implement decided strategy)
```

---

## Merge Conditions

Two stories may be merged if ALL of the following are true:

1. Combined task count is ≤ 7.
2. Both stories share the same primary entity.
3. The second story has the first as its only dependency.
4. No executor would be confused by the combined scope.

Example: "`<Entity>` Schema" (2 tasks) + "`<Entity>` Types" (2 tasks) → merge into
"`<Entity>` Data Foundation" (4 tasks).

---

## LOC Reference by Concern

These are estimates for task-level sizing (used by task-creator). Story
sizing uses task count, not LOC, but these help predict task count.

| Concern | Estimated LOC |
|---|---|
| Data model (3–6 fields) | 15–30 |
| DB migration (auto-generated) | — |
| TypeScript interface (entity) | 20–40 |
| Repository (4 CRUD methods) | 80–120 |
| Service (4 methods, validation) | 100–160 |
| Validator utility | 40–80 |
| API route (single method) | 50–100 |
| Custom hook | 60–100 |
| UI component (simple) | 60–120 |
| UI component (form) | 100–200 |
| Page | 40–80 |
| Unit test suite | 80–150 |

A story with 5 concerns at average LOC = ~500 LOC total. That is a normal sprint story.
A story over ~800 LOC is likely oversized and should fail Gate 1.
