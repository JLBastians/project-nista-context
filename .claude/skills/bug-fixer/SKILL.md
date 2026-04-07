---
name: bug-fixer
description: |
  STROBE bug fixing and test engineering skill. Use this skill whenever: debugging a bug, investigating unexpected behaviour, writing or running tests, reviewing code for correctness, triaging an error message or stack trace, verifying a fix works across happy path and edge cases, or documenting a resolved bug. Also trigger when the user mentions bug, error, broken, failing, test, regression, edge case, fix, debug, stack trace, crash, not working, or undefined behaviour. Trigger for both frontend (TypeScript/React) and backend (Python/FastAPI) code. This skill also manages the bug ledger files (Backend/BUGS.md and Frontend/BUGS.md) so that other agents can learn from past issues.
---

# STROBE Bug Fixer

You are debugging and testing code for STROBE, a fashion discovery app. Your job is to find the root cause of bugs, design tests that prove the fix works, and document what went wrong so the team doesn't repeat the same mistakes.

## Project Context

STROBE has two codebases — a Next.js frontend (`Frontend/`) and a FastAPI backend (`Backend/`). Bugs can live in either, or at the boundary between them (e.g., a frontend component expecting a response shape the API doesn't return). You need to be comfortable in both.

**Two-phase approach — know which phase the bug lives in:**
- **Current (PoC):** Mock data on frontend, scraped data on backend. Some bugs may be artefacts of mock data that won't exist in v0.
- **Target (v0):** Real Shopify data, live API integration. Bugs here are production-critical.

## Documentation Map

Read these as necessary when investigating:

| If you need to... | Read |
|------------|------|
| Understand expected behaviour for a feature | `Context/specs/` (relevant feature spec) |
| See the full user journey and flow logic | `Context/APP_FLOW.md` |
| Check database schema and column types | `Backend/api/README.md` Section 3 |
| Check API endpoint contracts | `Backend/api/README.md` Section 4 |
| Understand frontend component patterns | `Frontend/README.md` |
| See the backend project structure | `Backend/README.md` Section 3 |
| Check past bugs for patterns | `Backend/BUGS.md` and `Frontend/BUGS.md` |
| Understand interaction signal weights | `Context/ARCHITECTURE.md` (Interaction Signals) |

## Debugging Workflow

Follow this sequence for every bug. The goal is to understand before you fix — a wrong diagnosis leads to a patch that breaks something else.

### 1. Reproduce

Before touching any code, reproduce the bug reliably. Establish:
- **What's happening** — the actual behaviour (error message, wrong output, crash).
- **What should happen** — the expected behaviour (reference the relevant spec in `Context/specs/`).
- **Steps to reproduce** — the exact sequence of user actions or API calls.
- **Scope** — is this frontend-only, backend-only, or a boundary issue?

### 2. Isolate

Narrow down the root cause:
- **Frontend:** Check the component, hook, or state logic. Use browser console, React DevTools, network tab. Check if the issue is in the component rendering, the state management (`src/lib/store.tsx`), or the data it receives.
- **Backend:** Check the endpoint handler, CRUD function, or database query. Use the FastAPI `/docs` interactive docs to test endpoints in isolation. Check logs for stack traces.
- **Boundary:** If the frontend sends a request and gets unexpected data back, check both the Pydantic response schema (backend) and the TypeScript type definition (frontend `src/lib/types.ts`). Mismatches between these are a common source of bugs.

### 3. Write a Failing Test First

Before writing the fix, write a test that demonstrates the bug — a test that fails now and will pass once the fix is correct. This proves the fix actually addresses the root cause rather than a symptom.

**Backend tests (Python):**
```python
# tests/test_<module>_<bug_description>.py
def test_dismiss_interaction_logs_negative_weight():
    """Bug: dismiss events were logging weight 0.3 instead of -0.3."""
    interaction = create_interaction(type="dismiss", product_id=sample_id)
    assert interaction.weight == -0.3  # Must be negative
```

**Frontend tests (TypeScript):**
```typescript
// __tests__/<component>.<bug_description>.test.tsx
it('should revert heart icon on failed like API call', async () => {
  // Bug: heart stayed filled even when the API returned 500
  server.use(rest.post('/api/interactions', (req, res, ctx) => res(ctx.status(500))))
  render(<ProductCard product={mockProduct} />)
  await userEvent.click(screen.getByRole('button', { name: /like/i }))
  expect(screen.getByRole('button', { name: /like/i })).not.toHaveClass('text-red-500')
})
```

### 4. Fix

Apply the minimal fix that addresses the root cause. Avoid "while I'm here" changes — those belong in a separate task. The fix should be:
- **Targeted** — changes only what's necessary to fix the bug.
- **Safe** — doesn't introduce new edge cases or break existing behaviour.
- **Consistent** — follows the coding standards in the relevant skill (`frontend-engineer` or `backend-engineer`).

### 5. Verify

Run the test suite to confirm:
- **The failing test now passes** — the specific bug is fixed.
- **No existing tests broke** — no regressions introduced.
- **Edge cases are covered** — think about boundaries. If the bug was "negative weight not logged," also test zero weight, maximum weight, and missing weight.

**Backend:**
```bash
cd Backend
python -m pytest tests/ -v
```

**Frontend:**
```bash
cd Frontend
npm run lint
npm run build
```

### 6. Document in BUGS.md

After confirming the fix, add an entry to the appropriate bug ledger. This is how other agents learn from past issues.

- Backend bugs → `Backend/BUGS.md`
- Frontend bugs → `Frontend/BUGS.md`
- Boundary bugs → document in whichever side the root cause lived, with a cross-reference note.

## Bug Ledger Format

Each entry in `BUGS.md` follows this structure. The format is optimised for agents to scan quickly — they can search by symptom, root cause, or affected file.

```markdown
### BUG-<NNN>: <short description>

**Date:** YYYY-MM-DD
**Severity:** Critical / High / Medium / Low
**Symptom:** What the user or developer observed.
**Root cause:** Why it happened (the actual code/logic error).
**Files changed:** List of files modified in the fix.
**Fix:** Brief description of what was changed and why.
**Test:** Name of the test file/function that covers this bug.
**Lesson:** What to watch out for in future — the generalised takeaway.
```

**Example:**
```markdown
### BUG-001: Dismiss interaction logged with positive weight

**Date:** 2026-04-06
**Severity:** High
**Symptom:** Dismissed products kept reappearing in the For You feed because the recommendation engine treated them as positive signals.
**Root cause:** `create_interaction()` in `api/crud.py` used `abs(weight)` on all interactions, stripping the negative sign from dismiss events.
**Files changed:** `api/crud.py`, `tests/test_interactions.py`
**Fix:** Removed `abs()` wrapper; dismiss weight now passes through as -0.3 per ARCHITECTURE.md signal table.
**Test:** `test_dismiss_interaction_logs_negative_weight()` in `tests/test_interactions.py`
**Lesson:** Never normalise interaction weights at the storage layer — the sign carries semantic meaning (positive = liked, negative = disliked). Weight values should match ARCHITECTURE.md exactly.
```

The "Lesson" field is the most valuable part — it's what prevents the same class of bug from recurring. Write it as advice to a future developer, not just a description of what happened.

## Edge Case Checklist

When verifying a fix, consider these categories. Not all apply to every bug, but scanning the list catches things that ad-hoc testing misses.

**Data edge cases:**
- Empty / null / undefined values
- Zero vs missing (e.g., `price: 0` vs `price: null`)
- Very long strings (product titles, search queries)
- Special characters in user input
- Array with 0 items, 1 item, many items

**State edge cases:**
- Component unmounts during async operation
- User double-taps a button
- Network request fails mid-operation
- Stale data after navigation (back button)
- Cold start vs warm session

**Boundary edge cases:**
- API returns a different shape than TypeScript expects
- Database column type doesn't match Pydantic schema
- Frontend sends optional field; backend treats it as required

## Before You Start Debugging

1. Check `Backend/BUGS.md` or `Frontend/BUGS.md` — has this bug or something similar been fixed before?
2. Read the relevant spec in `Context/specs/` to confirm what the expected behaviour actually is.
3. Check if the bug is in PoC-phase code (mock data, scrapers) that won't exist in v0 — flag this if so.
4. If the bug involves interaction signals or taste vectors, verify weights against `Context/ARCHITECTURE.md`.
