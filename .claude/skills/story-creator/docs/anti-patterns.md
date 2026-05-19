# Story Anti-Pattern Catalogue

Ten enforced anti-patterns for story-creator. Check every story against all ten
before writing the file. A single violation requires remediation before proceeding.

---

## SANTI-001 — Giant Story

**Definition**: Story decomposes into more than 7 tasks after full decomposition.

**Symptoms**:
- Story goal mentions multiple user workflows ("create, view, edit, and delete")
- Affected Areas lists > 10 files
- Execution Strategy chain is more than 4 levels deep

**Remediation**: Apply Split Pattern 1 (By Layer) or Split Pattern 4 (By Component)
from `docs/sizing-guide.md`. A "full feature" always splits into at least a data
layer story and a UI story.

---

## SANTI-002 — Vague Goal

**Definition**: Story Goal fails the "newspaper test" — a non-engineer cannot understand
what changes for the user.

**Symptoms**:
- Goal contains implementation language: "implement", "create schema", "add repository",
  "set up", "configure", "refactor"
- Goal is passive or abstract: "`<Entity>` persistence is added", "The data layer exists"
- Goal has no subject: "Create a `<Entity>` model"

**Remediation**: Rewrite with an active user or system subject:
- ❌ "Implement `<Entity>` persistence layer"
- ✅ "Users can create `<entity>` records with a title and body that persist across sessions"
- ✅ "The system stores `<entity>` records in the database and retrieves them by ID"

---

## SANTI-003 — Cross-Domain Refactor

**Definition**: Story modifies the data layer of two or more primary entities
(see `.ai/architecture.md § Domain § Entities` for the list).

**Symptoms**:
- Affected Areas lists data schema changes for two different models
- Technical Decisions describes two unrelated entity relationships
- Repository files for two entities are in the same story

**Remediation**: One story per primary entity. Shared concerns (e.g., a junction
table linking two entities) go in one of the two entity stories — whichever
entity "owns" the relationship. Document the choice in Technical Decisions.

---

## SANTI-004 — Hidden Dependency

**Definition**: Story uses types, models, or functions produced by another story
that is not listed in the Dependencies section.

**Symptoms**:
- Affected Areas lists files that import from another story's output
- Execution Strategy references artifacts not yet created
- Task-creator generates tasks that depend on non-existent files

**Remediation**: Audit every file path in Affected Areas. For each file, ask:
"Does this file import anything produced by another story?" If yes, add that story
to Dependencies. Mark affected file paths as `[TBD: STORY-NNN]`.

---

## SANTI-005 — Architecture Drift

**Definition**: Technical Decisions section defers architectural choices to the executor.

**Symptoms**:
- Phrases like: "use whichever pattern is appropriate", "choose the best approach",
  "implement as you see fit", "follow conventions", "use standard patterns"
- Missing field: Technical Decisions section has sub-headings but no content
- Vague type names: "define the necessary types", "appropriate interfaces"

**Remediation**: Make every decision now. If the decision genuinely cannot be made
yet (pending spike, waiting for external input), add a Spike task as TASK-001,
block all other tasks on it, and document the unknown in Risks.

---

## SANTI-006 — Unconstrained Executor Scope

**Definition**: Affected Areas uses glob patterns, vague paths, or lists directories
instead of exact file paths.

**Symptoms**:
- "All `<entity>`-related files"
- "src/components/**"
- "any relevant configuration"
- "files as needed"
- Directory entries without file names

**Remediation**: List every file by exact path from the project root. For files
whose paths cannot be determined until an upstream task or story completes, write:
`[TBD: path determined by STORY-NNN task output]` and ensure the dependency is listed.

---

## SANTI-007 — Global Context Explosion

**Definition**: The story's context.md grows unbounded because executors append
unstructured content instead of following the append schema.

**Symptoms**:
- context.md contains full diff output
- context.md contains inline code blocks for entire file contents
- context.md entries have no structured headings (### Decisions, ### Files Changed)
- Execution Strategy does not reference the context update protocol

**Remediation**: Ensure the Execution Strategy section contains the context update
protocol reminder verbatim. Every task file (written by task-creator) MUST contain
a Context Update section with the exact append block. Executors append ONLY that
block — nothing else.

---

## SANTI-008 — Missing Story Acceptance Criteria

**Definition**: Story has no acceptance criteria, or criteria are subjective.

**Symptoms**:
- Acceptance Criteria section is empty or says "N/A"
- Criteria use: "works correctly", "looks good", "is complete", "functions as expected"
- Criteria are not runnable (no commands, no named outputs)

**Remediation**: Write ≥ 3 acceptance criteria. Always include:
1. The build command exits with code 0
2. A named export or curl command for the primary deliverable
3. A type-check assertion or runtime verification

---

## SANTI-009 — Premature UI

**Definition**: Story includes UI tasks before its data layer story is listed as a
dependency, or a UI story is sequenced before the API story it depends on.

**Symptoms**:
- UI story's Affected Areas includes components that call API routes not yet created
- UI story's Dependencies section is empty but Technical Decisions references API endpoints
- Execution Strategy starts with a component task before any API task exists

**Remediation**: Enforce the canonical dependency order:
```
Data Layer Story → API Layer Story → UI Story
```
UI stories MUST list their API story (or Data Foundation story) in Dependencies.
A hook that calls an API route CANNOT be in the same story as that route.

---

## SANTI-010 — Auth/Validation Coupling

**Definition**: Auth logic or shared validation utilities are embedded inside a domain
story instead of being isolated in their own story or task.

**Symptoms**:
- A domain story's Requirements mention "protect the route with auth middleware"
- A domain story creates a shared validator (used by multiple stories)
- Technical Decisions describe auth token strategy inline with entity logic

**Remediation**: Auth setup is always its own story (e.g., `STORY-NNN-user-auth-setup`).
Shared validators get their own story if they are used by multiple domain stories.
Domain stories reference auth/validator stories via Dependencies, not inline implementation.

---

## Red-Flag Phrases

The following phrases in any story section are automatic violations.
Find and remove them before writing the story file.

| Phrase | Violation |
|---|---|
| "implement X" (as a goal) | SANTI-002 |
| "set up X" (as a goal) | SANTI-002 |
| "add X as needed" | SANTI-005, SANTI-006 |
| "appropriate pattern" | SANTI-005 |
| "best practices" | SANTI-005 |
| "as you see fit" | SANTI-005 |
| "relevant files" | SANTI-006 |
| "any related" | SANTI-006 |
| "works correctly" | SANTI-008 |
| "looks good" | SANTI-008 |
| "fully implemented" | SANTI-008 |
| "handle X" (without specifics) | SANTI-002, SANTI-005 |
| "refactor" (in a story goal) | SANTI-002 |
| "improve" (without measurable outcome) | SANTI-002 |
| "protect the route" (without naming auth strategy) | SANTI-010 |
| "use appropriate auth" | SANTI-010 |
| "validate input" (without naming validator) | SANTI-010 |
