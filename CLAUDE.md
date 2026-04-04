# Instructions for Claude

You are assisting with Project Nista (brand name: **STROBE**), a fashion discovery app in early-stage development. The founder is non-technical — write clean, well-commented code and explain decisions clearly. When unsure about a decision, ask rather than assume.

**Key rules:**
- Never hardcode secrets or API keys — always use environment variables
- Backend: Follow PEP 8, use Pydantic for validation, SQLAlchemy for DB operations
- Frontend: Follow ESLint rules, use TypeScript strictly (no `any`), Tailwind for styling
- Scrapers: Respect rate limits (5s delay between requests), handle errors gracefully, log failures
- Mobile-first: Design for 375px viewport, test responsive behaviour
- Always explain changes and ask for permission before making direct updates to production files
- Git branching: `main` branch for Frontend contains the landing page (live, connected to backend API on Railway). `dev` branch contains the web app under active development (running on mock data, not yet connected to backend). Always confirm which branch you're on before making changes.
- Use transactions for database writes
- Handle token expiration gracefully on frontend
- Add loading states for all async operations

---

# Fashion Discovery App — Project Overview

## Project Purpose
A fashion discovery app that helps users find apparel from Australian retailers through AI-powered recommendations. The system learns user preferences through interactions (liking/disliking products, viewing product details, searching) to show increasingly tailored suggestions.

## Core Features
1. **User Interface**: Users will see multiple products on the page in cards, with buttons to like / dislike / add to cart; or can tap to go to the product page. Potential for a Tinder-style like/dislike on apparel products
2. **AI-Powered Matching**: Learns user preferences via product tags, like/dislike history and other interactions to show more relevant products over time
3. **Multi-Profile Support**: Users can create multiple style profiles (e.g., office, casual, weekend). The system will learn and differentiate preferences across profiles, showing tailored recommendations for each
4. **Retailer Data**: Product data scraped from 2-3 US retailers and 2-3 Australian retailers (e.g., The Iconic, Uniqlo AU, Cotton On)

## Development Phase
Currently in **PoC stage**: Building proof of concept targeting ~10,000 products from 3–4 retailers. Currently ~4,000 Myer products scraped and fully embedded (both models); Revolve scraper operational (dresses category — multi-category expansion in progress). Basic matching and core user interaction functionality being built.

---

## Repository Structure

```
project-nista/
├── Backend/
│   ├── api/              # FastAPI app (main.py, models.py, crud.py, database.py, README.md)
│   ├── model/            # Embedding model adapters, pipeline scripts, and docs
│   │   ├── adapters/     # BaseEmbedder ABC, Marqo adapter, Qwen adapter, factory
│   │   ├── run_pipeline.py  # Launcher: routes pipeline commands to correct model venv
│   │   └── README.md     # Full pipeline documentation
│   ├── scripts/          # Pipeline scripts (bulk_embed_catalog, incremental_embed, integrity test)
│   │   └── eval/         # Evaluation tools (similarity_inspector.py, model_comparison.py)
│   ├── scrapers/         # Product scrapers + shared helpers
│   │   ├── db_helpers.py    # Shared DB operations (insert, dedup, update-details, JSON backup)
│   │   ├── myer_scraper.py  # Myer AU scraper (Next.js __NEXT_DATA__ extraction)
│   │   ├── revolve_scraper.py  # Revolve AU scraper (AJAX catalog API + curl_cffi)
│   │   └── README.md     # Scraper documentation
│   ├── utils/            # Helper functions
│   ├── data/             # Runtime data (gitignored where appropriate)
│   │   ├── raw/          # Raw scraped data (JSON backups)
│   │   └── eval/         # Cached vectors and HTML reports
│   ├── tests/            # Test and diagnostic scripts
│   ├── README.md         # Backend setup and environment guide
│   └── requirements.txt  # API dependencies (project-nista-env)
├── Frontend/
│   ├── src/
│   │   ├── app/          # Next.js App Router pages
│   │   │   ├── (app)/    # Authenticated app pages (feed, search, wishlist, product, profile, etc.)
│   │   │   ├── (auth)/   # Auth pages (login, register, reset-password, welcome)
│   │   │   └── (onboarding)/  # Onboarding flow
│   │   ├── components/   # Shared components + shadcn/ui primitives
│   │   ├── hooks/        # Custom hooks (use-mobile, use-toast)
│   │   ├── lib/          # Utilities, types, mock data, app state (store.tsx)
│   │   └── styles/       # Global CSS
│   ├── package.json
│   └── tsconfig.json
└── CLAUDE.md             # This file
```

---

## Tech Stack

### Frontend
**Landing page (main branch):** *(implemented — live on Vercel, connected to backend API)*
**Web app (dev branch):** *(implemented — under active development, running on mock data, not yet connected to backend)*
- **Framework**: Next.js 16 (App Router) with React 19 and TypeScript
- **Styling**: Tailwind CSS v4 (CSS-first config via `@import 'tailwindcss'`)
- **UI Components**: shadcn/ui (Radix UI primitives)
- **State Management**: React Context + useReducer (`src/lib/store.tsx`)
- **Forms**: react-hook-form + Zod validation
- **Notifications**: Sonner (toast notifications)
- **Icons**: Lucide React
- **Font**: Clash Grotesk via Fontshare CDN
- **Analytics**: Vercel Analytics (`@vercel/analytics`)
- **Hosting**: Vercel

### Backend *(implemented — API and database live, scraper operational)*
- **Framework**: FastAPI (Python 3.12), served via Uvicorn
- **ORM**: SQLAlchemy 2.0
- **Database**: PostgreSQL 18 with pgvector 0.8.2 (local `strobe_dev` on localhost:5432 for development; Neon for production/live API)
- **Database Client**: psycopg2-binary
- **Vector Search**: pgvector (768-dim Marqo + 2048-dim Qwen embeddings for product similarity)
- **Data Processing**: pandas, numpy
- **Web Scraping**: BeautifulSoup4 + lxml + requests (Myer — Next.js `__NEXT_DATA__` extraction); curl_cffi with Chrome TLS impersonation (Revolve — bypasses bot detection at TLS layer)
- **Config**: python-dotenv
- **Hosting**: Railway.app (production API reads from Neon; scrapers and embedding pipeline use local DB)

### Authentication *(planned — not yet implemented)*
- **Provider**: Clerk (free tier: 10k monthly active users)
- Protected API endpoints will use Clerk JWT validation via middleware
- Frontend will integrate Clerk's React SDK

### Embedding Models *(implemented — two models operational, full catalog embedded)*
- **Marqo-FashionSigLIP**: 768-dim fashion-specific embeddings (default model)
- **Qwen3-VL-Embedding-2B**: 2048-dim general-purpose VLM embeddings (second model for comparison)
- **Pipeline**: Model-agnostic adapter system with per-model venvs and `model/run_pipeline.py` launcher
- **Image Hosting**: Direct URLs from retailer sites (no CDN for PoC)
- See `Backend/model/README.md` for full pipeline documentation

### AI Tagging *(planned — not yet implemented)*
- Model TBD — candidates include Qwen-VL, LLaVA, GPT-4o, or zero-shot classification with Marqo-FashionSigLIP

---

## System Architecture

### Data Flow *(partially implemented — scrapers and embedding pipeline are live; AI tagging planned)*
```
Retailers → Scraper → PostgreSQL (local dev) → Embedding Pipeline → FastAPI (Railway/Neon) → React Frontend
                           ↓ (optional)                ↓
                     JSON backup (data/raw/)    vectors written back to DB
```

Scrapers insert products directly into the local PostgreSQL database (no intermediate JSON required — JSON is just an optional backup). The embedding pipeline then reads unembedded products from the DB, downloads images, runs them through the model, and writes vectors back. The Railway-hosted FastAPI reads from the Neon production database. To sync local data to Neon, use `pg_dump` / `pg_restore`.

For planned systems (AI tagging, matching algorithm, future features), see `Backend/TODO.md`.

---

## README Index

When you need to understand how a subsystem works, read the relevant README before making changes. These are the project's README files:

| README | Covers |
|--------|--------|
| `Backend/README.md` | Backend setup, environment configuration, venv structure, getting started |
| `Backend/api/README.md` | Database schema (all models and columns), API endpoints, request/response formats |
| `Backend/model/README.md` | Embedding pipeline architecture, adapter system, `run_pipeline.py` commands, per-model venvs |
| `Backend/scrapers/README.md` | Scraper architecture, `db_helpers.py` shared functions, adding new retailers |
| `Backend/attribute-extraction/README.md` | Style attribute extraction tool, taxonomy, enrichment pipeline, text prompt generation |
| `Backend/TODO.md` | Prioritised task list with detailed implementation notes for pending work |

---

## Documentation Update Protocol

After completing any task that changes functionality, adds/removes files, or modifies interfaces, run through this checklist before marking the work done:

1. **List changed files.** Enumerate every file created, modified, or deleted in this task.
2. **Update nearby READMEs.** For each changed file, check if its directory (or parent directory) has a README.md. If so, update it to reflect the changes — especially command examples, file descriptions, column names, and architecture notes. See the [README Index](#readme-index) below for the full list.
3. **Create new READMEs where appropriate.** If you created a new directory or standalone tool, consider whether it needs its own README.md. Don't create one for trivial utility folders — only for directories that a developer would need orientation to understand.
4. **Update TODO.md.** If the completed task corresponds to a TODO item (`Backend/TODO.md`), mark it done with a brief note of what was completed. If new follow-up tasks were discovered during implementation, add them.
5. **Update CLAUDE.md (structural changes only).** Check if the repository structure, tech stack, or key decisions sections need updating. Skip this for non-structural changes.
6. **Cross-check consistency.** Verify that any commands, file paths, column names, or environment variables mentioned in updated documentation match the actual code. This is the most common source of doc drift.

---

## Development Constraints

1. **Budget-Conscious**: Minimise API costs, use free tiers where possible
2. **Legal Compliance**: Respect retailer ToS, robots.txt, rate limits (1 req/5sec)
3. **Non-Technical Founder**: Code should be well-documented, follow standard patterns
4. **Mobile-First**: Responsive design, test on 375px viewport
5. **PoC Scope**: Start simple, defer complex or nice-to-have features

