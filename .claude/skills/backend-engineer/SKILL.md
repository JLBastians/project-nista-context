---
name: backend-engineer
description: |
  STROBE backend engineering skill for building and maintaining the FastAPI backend, database schema, API endpoints, scrapers, and ingestion pipeline. Use this skill whenever working on: API endpoints, database models or migrations, CRUD operations, scraper development or debugging, Trigger.dev ingestion jobs, Shopify Storefront API integration, PostgreSQL/Neon database operations, SQLAlchemy models, Pydantic validation schemas, backend error handling, or any Python code under Backend/. Also trigger when the user mentions API, endpoint, database, schema, migration, scraper, ingestion, Neon, Railway, FastAPI, or references Backend/ files. Trigger for cart/checkout backend logic (Shopify cart API, cart_events logging), user interaction logging endpoints, or any data pipeline / enrichment / vector embedding work.
---

# STROBE Backend Engineer

You are building the backend for STROBE, a fashion discovery app. This document contains the conventions, architecture, and patterns you need to write consistent, production-quality backend code.

## Project Context

STROBE's backend is a FastAPI application (Python 3.12) currently deployed on Railway, with PostgreSQL (local `strobe_dev` for development, Neon for production) as the database. The backend serves product data, user profiles, interaction logging, and cart operations to the Next.js frontend.

**Two-phase approach — know which phase you're building for:**
- **Current (PoC):** Custom scrapers (Myer, Revolve) → local PostgreSQL → FastAPI on Railway. Used for enrichment testing.
- **Target (v0):** Shopify tokenless Storefront API → Trigger.dev ingestion → Neon PostgreSQL → same FastAPI endpoints.

This skill covers the API layer, database, scrapers, and ingestion + enrichment pipeline.

## Documentation Map

Read these before writing code as necessary:

| If you need to... | Read |
|------------|------|
| Understand the tech stack | `Backend/README.md` Section 2 |
| Understand the backend directory structure | `Backend/README.md` Section 3 |
| Understand the virtual environment architecture | `Backend/README.md` Section 4 |
| Understand the full schema (all tables, columns, types) | `Backend/api/README.md` Section 3 |
| See all API endpoints and request/response formats | `Backend/api/README.md` Section 4 |
| Understand scraper architecture | `Backend/scrapers/README.md` |
| Understand the embedding pipeline interface | `Backend/model/README.md` |
| See how the frontend will consume your endpoints | `Context/specs/` (relevant feature spec) |
| Understand the overall system architecture | `Context/ARCHITECTURE.md` |

## Coding Standards

### Python Style

- Follow PEP 8. Use type hints on all function signatures.
- Use `pathlib.Path` over `os.path`.
- Use f-strings for string formatting.
- Docstrings on all public functions (one-liner or Google-style).
- Keep functions focused — if a function exceeds ~40 lines, consider splitting.

### Pydantic Models

Every API endpoint uses Pydantic models for both request and response validation. This is non-negotiable — it provides automatic documentation, type safety, and clear contracts with the frontend.

```python
from pydantic import BaseModel, Field
from datetime import datetime
from uuid import UUID

class ProductResponse(BaseModel):
    id: UUID
    title: str
    brand_name: str
    price: float
    sale_price: float | None = None
    image_urls: list[str]
    occasion_tags: list[str] = Field(default_factory=list)

    model_config = {"from_attributes": True}
```

### SQLAlchemy Patterns

- All models in `api/models.py` using SQLAlchemy 2.0 declarative style.
- Use `Mapped[]` type annotations on all columns.
- Always use transactions for writes — wrap in `with db.begin():` or use the session's transaction context.
- Use `select()` statement style (SQLAlchemy 2.0), not the legacy `query()` API.

```python
from sqlalchemy import select

# Good: 2.0 style
stmt = select(Product).where(Product.brand_id == brand_id)
result = db.execute(stmt).scalars().all()

# Bad: legacy query style
products = db.query(Product).filter_by(brand_id=brand_id).all()
```

### Database Operations

- All CRUD operations live in `api/crud.py` — endpoints call crud functions, never write SQL directly in route handlers.
- Use database transactions for any write operation. If multiple tables need updating together (e.g., logging an interaction + updating a taste vector), wrap them in a single transaction.
- For bulk inserts (scraper data, batch embeddings), use `bulk_save_objects()` or `executemany()` for performance.
- Always handle `IntegrityError` for unique constraint violations (duplicate products, duplicate interactions).

### API Endpoint Patterns

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from api.database import get_db

router = APIRouter(prefix="/products", tags=["products"])

@router.get("/{product_id}", response_model=ProductResponse)
def get_product(product_id: UUID, db: Session = Depends(get_db)):
    product = crud.get_product(db, product_id)
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return product
```

- Use dependency injection (`Depends(get_db)`) for database sessions.
- Return appropriate HTTP status codes: 200 (success), 201 (created), 400 (bad request), 404 (not found), 422 (validation error — Pydantic handles this), 500 (server error).
- Add loading states context: if an endpoint is slow, document why and whether the frontend should show a loading indicator.

### Environment Variables

Never hardcode secrets. All configuration through environment variables:

```python
import os
DATABASE_URL = os.environ["DATABASE_URL"]  # Required — fail fast if missing
ALLOWED_ORIGINS = os.environ.get("ALLOWED_ORIGINS", "http://localhost:3000")
```

Use `.env` locally (loaded by the runner), `.env.example` as the template. Required vars fail with a clear error if missing.

### Scraper Standards (PoC Phase)

Scrapers are temporary (replaced by Shopify API in v0) but still need quality:

- **5-second delay** between requests — respect retailer servers.
- **Error handling on every request** — catch timeouts, 403s, rate limits. Log and continue.
- **Log all failures** — product URL, error type, timestamp. Never silently skip.
- **Deduplication** — check `product_url` uniqueness before inserting. Use `db_helpers.py` shared functions.
- **JSON backup** — save raw scraped data to `data/raw/` before DB insert.

### Shopify Storefront API (Target v0)

The production data source. Key things to know:

- **No auth required** — public GraphQL endpoint at `https://{store}.myshopify.com/api/2024-01/graphql.json`.
- **Query complexity limit:** 1,000 units per request (~20-30 products with moderate fields).
- **Data is ephemeral** — Shopify's terms prohibit permanent warehousing. Products have `fetched_at` and `expires_at` timestamps. Stale products are re-fetched on the 4–6 hour cycle.
- **Enrichment trigger:** New or updated products get `enrichment_status = 'pending'`. Josh's pipeline picks these up.

### Interaction Logging

The frontend sends interaction events — the backend must log them reliably because the recommendation engine depends on this data:

- Use a transaction to ensure both writes succeed or neither does.
- Never drop interaction data silently — if a write fails, return an error so the frontend can retry.

### Testing

- Write tests for CRUD operations and critical endpoints.
- Use a test database (separate from dev) or SQLAlchemy's in-memory SQLite for unit tests where pgvector isn't needed.

## Before You Start Coding

1. Read the relevant documentation laid out in the Documentation Map.
2. If the feature has a frontend spec (in `Context/specs/`), read the relevant frontend spec to understand what the frontend expects.
3. Check `api/crud.py` for existing operations you might reuse.
4. Check `api/models.py` for existing table definitions.
5. Run the API locally to test: `uvicorn api.main:app --reload`

## After You Finish Coding

1. Go to C:\Users\AvantiGomes\Development\.claude\changelog\backend and check to see if there is an existing changelog file with the current date. If not, create a new file named <yyyymmdd>.md`.
2. Document changes you made in the changelog file with the following format:
	- `[Component/Feature Name] - [Short description of the change]`
	- `[Request] - [Overview of what was asked and reason why if available]`
	- `[Solution] - [Description of the change]`
	- `[Files impacted] - [List any files that were updated]`
	- `[Test] - [Summary of test outcomes]`