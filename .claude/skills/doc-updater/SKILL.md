---
name: doc-updater
description: |
  STROBE documentation updater skill for keeping markdown files consistent and accurate after code changes. Use this skill whenever: a feature has been built or modified and docs need updating, a new file or endpoint has been added, a spec needs to reflect a design decision, the user asks to update docs or propagate a change across markdown files, or you notice documentation has drifted from code reality. Also trigger when the user mentions docs, documentation, update readme, update spec, propagate changes, sync docs, markdown, or references any .md file in Context/ or any README.md. Trigger after completing a coding task that changed interfaces, added files, or modified behaviour — documentation should never be stale.
---

# STROBE Doc Updater

You are maintaining STROBE's documentation system. Your job is to keep every markdown file accurate, consistent, and free of duplication after code or design changes. Stale docs mislead other agents and waste development time.

## Project Context

STROBE's documentation is organised in layers. Each layer has a specific purpose and audience. Understanding the hierarchy prevents duplication — the core principle is **each fact lives in ONE place**.

**Documentation hierarchy (most general → most specific):**

1. `CLAUDE.md` — Agent entry point. Project identity, coding rules, documentation map. Only updated when the doc map itself changes.
2. `Context/` — Product and design docs. What STROBE is, how it's built, what to build next.
3. `Context/specs/` — Feature-level functional requirements. One file per feature.
4. `Backend/README.md` and `Frontend/README.md` — Engineering setup, structure, and how to run each codebase.
5. `Backend/api/README.md`, `Backend/model/README.md`, `Backend/scrapers/README.md`, `Backend/attribute-extraction/README.md` — Deep technical references for specific subsystems.
6. `Backend/BUGS.md` and `Frontend/BUGS.md` — Bug ledgers (managed by the bug-fixer skill).
7. `Backend/TODO.md` — Task backlog and priorities.

## Documentation Map

| If you need to... | Read |
|------------|------|
| See which doc owns which facts | `CLAUDE.md` Documentation Map table |
| Understand the doc update protocol | `CLAUDE.md` Documentation Update Protocol |
| See the full doc index and relationships | `Context/README.md` |
| Check what decisions are resolved vs open | `Context/DECISIONS.md` |

## Update Protocol

This is the core workflow. Follow it after any task that changes functionality, adds or removes files, or modifies interfaces.

### 1. Identify What Changed

Start by listing every file that was created, modified, or deleted. Be explicit — don't summarise. For example:
- `Backend/api/crud.py` — added `get_interactions_by_session()` function
- `Backend/api/models.py` — added `session_id` column to `user_interactions` table
- `Frontend/src/components/occasion-filter.tsx` — new component

### 2. Map Changes to Documents

Each type of change has a natural home in the doc system. Use this mapping:

| What changed | Primary doc to update | Secondary docs to check |
|-------------|----------------------|------------------------|
| New/modified API endpoint | `Backend/api/README.md` | Feature spec in `Context/specs/` that consumes it |
| Database schema change | `Backend/api/README.md` Section 3 | `Context/ARCHITECTURE.md` if it affects the schema overview |
| New frontend component or page | `Frontend/README.md` (project structure) | Feature spec in `Context/specs/` |
| Feature behaviour change | Relevant `Context/specs/[feature].md` | `Context/APP_FLOW.md` if user flow changed |
| System architecture change | `Context/ARCHITECTURE.md` | `Context/ROADMAP.md` if phase goals affected |
| New design decision | `Context/DECISIONS.md` | Relevant spec if the decision affects a feature |
| Priority or scope change | `Context/PRODUCT_BRIEF.md` | `Context/ROADMAP.md`, affected specs |
| Task completed | `Backend/TODO.md` | — |
| New task discovered | `Backend/TODO.md` | — |
| Bug fixed | `Backend/BUGS.md` or `Frontend/BUGS.md` | — |
| New file added to project | Nearest directory README | — |

### 3. Make the Updates

For each document identified in Step 2:

- **Read the current version first** — understand the existing structure before editing.
- **Update in place** — modify the specific section that's affected. Don't rewrite entire documents.
- **Preserve the document's voice and format** — match the heading style, table format, and level of detail that already exists.
- **Add, don't duplicate** — if a fact already exists in another document, reference it rather than restating it. Use cross-references like "See `Backend/api/README.md` Section 3 for full schema."
- **Mark version context** — if something is PoC-only or v0-target, say so explicitly.

### 4. Ripple Check

After updating the primary document, scan for ripple effects — places where the change might invalidate existing content:

- **Search for the old value** — if you renamed something, search all `.md` files for the old name.
- **Check cross-references** — if you changed a section heading, check if other docs reference it by name (e.g., "See ARCHITECTURE.md Section 7").
- **Verify file paths** — if you moved or renamed a file, check all docs that reference its path.
- **Check consistency of numbers** — if a count changed (e.g., "5-step onboarding" was previously "6-step"), search all docs for the old number.

```bash
# Useful commands for ripple checking
grep -r "old_term" Context/ Frontend/README.md Backend/README.md --include="*.md"
grep -r "6-step" Context/ --include="*.md"
```

### 5. Cross-Check

Final verification pass. For each updated document, confirm:

- [ ] File paths mentioned in the doc actually exist in the codebase.
- [ ] Column names mentioned match `Backend/api/README.md` schema.
- [ ] Environment variable names match `.env.example`.
- [ ] Feature priority labels (P0, P1, P2) match `Context/PRODUCT_BRIEF.md`.
- [ ] No duplicate facts across documents (each fact lives in ONE place).

## Writing Standards

### Tone

Technical, precise, concise. Write for an AI agent that will read this doc before writing code. Every sentence should either inform a decision or prevent a mistake.

### Structure

- Use tables for structured data (schemas, mappings, comparisons).
- Use code blocks for file paths, commands, and code snippets.
- Use blockquotes (`>`) for notes, caveats, and P1/P2 deferral callouts.
- Keep paragraphs short — 2–3 sentences maximum.

### What Not to Do

- Don't add preambles or summaries that restate the heading.
- Don't duplicate content that lives in another document — cross-reference instead.
- Don't add speculative content ("In the future, we might...") unless it's in a clearly labelled "Future" or "Open Decisions" section.
- Don't update `CLAUDE.md` unless the documentation map itself needs changing or the user explicitly requests it.
- Don't touch any Backend `.md` files unless specifically asked to (the backend engineer owns those).

## Common Update Scenarios

### New API endpoint added

1. Add endpoint to `Backend/api/README.md` Section 4 (method, path, request/response schema, description).
2. If a frontend spec will call this endpoint, add a note in the relevant `Context/specs/[feature].md` referencing the endpoint.

### Database column added or changed

1. Update the schema table in `Backend/api/README.md` Section 3.
2. If the column affects the architecture overview (e.g., a new vector type), update `Context/ARCHITECTURE.md`.

### Feature spec changed (e.g., onboarding step count changed)

1. Update the primary spec file in `Context/specs/`.
2. Search all other `.md` files for references to the old value.
3. Update `Context/APP_FLOW.md` if the user journey changed.
4. Update `Context/ROADMAP.md` if phase tasks or exit criteria reference it.
5. Update `Context/DECISIONS.md` if this resolves or opens a decision.
6. Check `Context/README.md` spec table descriptions.

### Design decision made

1. Add or move the decision in `Context/DECISIONS.md` (Open → Resolved).
2. Update the relevant spec or architecture doc to reflect the decision.

### Priority tier changed (e.g., feature moved from P0 to P1)

1. Update `Context/PRODUCT_BRIEF.md` feature summary.
2. Update the relevant spec with a priority banner or note.
3. Update `Context/ROADMAP.md` phase tasks if ownership or timeline changed.

## Before You Start Updating

1. Read `CLAUDE.md` Documentation Update Protocol — this is the canonical process.
2. Understand which document owns the fact you're updating (use the mapping table above).
3. Read the target document's current version before editing.
4. After editing, run a ripple check to catch cascading impacts.
