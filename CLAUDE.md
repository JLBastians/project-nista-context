> v3 · April 2026

# STROBE — Agent Entry Point

## Project Identity

**STROBE** is a fashion discovery app (internal codename: Project Nista) built by a non-technical founder. Early-stage PoC/v0 targeting US market launch via Shopify product data and Vercel Next.js web app. The application enables an AI-powered feed or "For You" page of fashion products tailored to each user's taste vector, built through a 5-step onboarding flow (age, gender expression, city, lifestyle activities, brand selections) and reinforced through explicit signals (likes, dismisses, wishlist adds, add-to-bag) and passive signals (dwell time, product detail views). A swipe mechanic for deeper taste vector seeding is planned as a P1 enhancement. Taste vectors are maintained per occasion (everyday, going out, work, special).

---

## Development Phase

**Current PoC Phase (Local + Railway):**
- Backend: FastAPI on Railway, local PostgreSQL + Neon DB production + local database mirroring Neon DB for enrichmnent testing, Marqo-FashionSigLIP + Qwen embeddings
- Frontend: Next.js on Vercel — `main` branch has live landing page (strobelab.co), `dev` branch has web app in development (mock data, not yet connected to API)
- Data: Currently scraped from Myer and Revolve for enrichment testing. Scraped data is temporary — Shopify tokenless Storefront API will ultimately be the production source.

**Target v0 Architecture:**
- Data source: Shopify tokenless Storefront API (no auth required)
- Ingestion: Trigger.dev (or similar) background jobs (every 4–6 hours)
- Database: Neon serverless PostgreSQL with pgvector
- Enrichment: Qwen (tagging), Marqo (vector search) is the current approach (TBD effectiveness)
- Cart & checkout: Shopify tokenless cart API handoff to hosted Shopify checkout
- Auth: TBD (NextAuth.js or Clerk — decide before Phase 4)
- Analytics: TBD (before Phase 2)

---

## Coding Rules

**All languages:**
- Never hardcode secrets — always use environment variables
- Explain changes and ask permission before updating production files
- Add loading states for all async operations

**Backend (Python):**
- Follow PEP 8, use Pydantic for validation, SQLAlchemy for DB operations
- Use transactions for database writes
- Scrapers: 5s delay between requests, error handling, log all failures

**Frontend (TypeScript/React):**
- Strict TypeScript (no `any`)
- ESLint rules, Tailwind for styling, mobile-first (375px viewport)
- React Context + useReducer for state management

---

## Documentation Map

**Read these files when you need to...**

| File | Purpose |
|------|---------|
| **Context/PRODUCT_BRIEF.md** | Understand problem statement, personas, goals, success metrics |
| **Context/ARCHITECTURE.md** | See system design, tech stack, data flow, complete schema |
| **Context/ROADMAP.md** | Find build phases, task breakdown, timeline, ownership |
| **Context/DECISIONS.md** | Check what decisions have been made or are still open |
| **Context/APP_FLOW.md** | Trace the complete user journey and screen inventory |
| **Context/specs/[feature].md** | Deep dive into a specific front-end feature's requirements (e.g. checkout) |
| **Backend/README.md** | Backend setup, venvs, running the API |
| **Backend/api/README.md** | Database schema, all tables/columns, API endpoints, request/response formats |
| **Backend/model/README.md** | Embedding pipeline architecture, `run_pipeline.py` commands, per-model venvs |
| **Backend/scrapers/README.md** | Scraper architecture, adding new retailers |
| **Backend/attribute-extraction/README.md** | Style attribute extraction, taxonomy, enrichment pipeline |
| **Backend/TODO.md** | Current task backlog and priorities |

---

## Documentation Update Protocol

After any task that changes functionality, adds/removes files, or modifies interfaces:

1. **List changed files** — enumerate every file created, modified, or deleted
2. **Update the nearest relevant doc:**
   - Code change → update its directory's README.md
   - Feature behaviour change → update the feature's spec file in Context/specs/
   - System design change → update Context/ARCHITECTURE.md
3. **Update Backend/TODO.md** if a task was completed or new tasks discovered
4. **Update this CLAUDE.md only** if documentation map or if specifically requested by the user
5. **Cross-check** — verify file paths, column names, env vars in docs match actual code

Key principle: **each fact lives in ONE place.** Don't duplicate across docs.

---

## Development Constraints

- **Budget-conscious** — minimise API costs, use free tiers where possible
- **Legal compliance** — respect retailer ToS, robot