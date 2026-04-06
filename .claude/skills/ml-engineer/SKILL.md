---
name: ml-engineer
description: |
  STROBE ML/AI engineering skill for the enrichment pipeline, embedding models, taste vectors, and personalisation engine. Use this skill whenever working on: Marqo-FashionSigLIP embeddings (768-dim product vectors), Qwen visual tagging, style attribute extraction (67-dim style vectors), the embedding adapter system, taste vector calculation or session recalculation, personalisation engine logic, feed ranking algorithms, cold start handling, onboarding vector seeding, model evaluation or comparison, or any code under Backend/model/ or Backend/attribute-extraction/. Also trigger when the user mentions embeddings, vectors, Marqo, Qwen, enrichment, tagging, taste profile, personalisation, recommendation engine, similarity search, vector search, pgvector, cosine similarity, or FashionSigLIP. Trigger for debugging embedding quality, model comparison reports, or batch embedding pipeline issues.
---

# STROBE ML Engineer

You are building the AI enrichment and personalisation layer for STROBE, a fashion discovery app. This document covers the embedding pipeline, model architecture, taste vector system, and personalisation engine.

## Project Context

STROBE uses AI at two levels: (1) product enrichment — transforming raw catalog data into tagged, vectorised products; and (2) personalisation — learning each user's style preferences and ranking products accordingly. Both are critical to the core product experience.

**Ownership context:** This is Josh's domain. Avanti builds the frontend UI that consumes the outputs (feed, search results, occasion filters). The interface between domains is clean: enrichment writes to specific database tables (`product_tags`, `product_embeddings`, `style_attributes`), and the personalisation engine exposes taste vectors that the feed queries.

## Documentation Map

| Need to... | Read |
|------------|------|
| Understand the embedding pipeline architecture | `Backend/model/README.md` |
| See adapter system, model comparison, pipeline commands | `Backend/model/README.md` |
| Understand style attribute extraction | `Backend/attribute-extraction/README.md` |
| See the full database schema (embedding columns, tables) | `Backend/api/README.md` Section 3 |
| Understand how the feed consumes vectors | `Context/specs/feed.md` |
| Understand how search uses product vectors | `Context/specs/search.md` |
| See the overall system architecture | `Context/ARCHITECTURE.md` Layers 3–4 + Personalisation |
| Understand onboarding vector seeding | `Context/specs/onboarding.md` |

## The Two Vector Types

This is the most important concept in the system. STROBE uses two distinct vector representations, each optimised for a different purpose:

### 1. Product Vector (Marqo-FashionSigLIP, 768-dim)

- **What it encodes:** Visual and textual similarity *within* a product category.
- **Used for:** Search (semantic nearest-neighbour), "find similar items" in browse mode, product-to-product similarity.
- **Example:** Searching "navy blazer" returns other blazers ranked by visual/textual similarity.
- **Storage:** `product_embeddings` table → vectors stored in Marqo index, document IDs in `marqo_document_id` column.
- **DB columns:** `embedding_vector_marqo` (768-dim) + `embedded_at_marqo` on the `products` table.

### 2. Style Vector (Zero-shot FashionSigLIP, 67-dim)

- **What it encodes:** Structured style attributes (waist fit, sleeve length, neckline, etc.) and broader cross-category aesthetic similarity.
- **Used for:** Discovery and personalisation in the For You feed. Enables finding aesthetic coherence across *different* product categories.
- **Example:** A user who likes minimalist linen shirts also sees minimalist trousers, bags, and outerwear — because their style vectors are similar even though the product categories are different.
- **Storage:** `style_attributes` JSONB column on products + `style_vector` for the 67-dim representation.
- **Why 67 dimensions:** Each dimension corresponds to a specific style attribute classifier (silhouette, neckline, waist, sleeve length, material, colour tone, aesthetic, etc.).

**The key insight:** Search uses product vectors (intra-category precision). The For You feed uses style vectors (cross-category aesthetic matching). Taste vectors — the per-user representations — are computed from style vectors, not product vectors, because personalisation needs to work across categories.

## Embedding Pipeline Architecture

The pipeline is model-agnostic with an adapter system:

```
Pipeline Scripts (bulk_embed, incremental_embed)
        │
        ▼ calls standard interface
Adapter Layer (model/adapters/)
    ├── BaseEmbedder (abstract)
    │   ├── load()
    │   ├── embed_images_batch(images) → np.ndarray
    │   ├── preprocess(image) → Image
    │   ├── embedding_dim → int
    │   ├── model_name → str
    │   ├── db_column → str
    │   └── db_timestamp_column → str
    │
    ├── MarqoFashionSigLIP (768-dim, fashion-specific)
    └── QwenVL (2048-dim, general VLM)

EmbedderFactory.create("marqo") → ready-to-use adapter
```

**Key design decisions:**
- Each model writes to its own DB columns (`embedding_vector_<model>` + `embedded_at_<model>`). Vectors from different models coexist — no overwriting.
- Pre-processing (grayscale, background removal) is pipeline-owned, not model-owned. The adapter only handles model-specific transforms.
- The pipeline scripts call `embedder.embed_images_batch()` and get normalised numpy arrays back. All model-specific logic is hidden inside the adapter.

### Running the Pipeline

All commands from the `Backend/` directory via `model/run_pipeline.py`:

```bash
# Bulk embed with Marqo (default)
python model/run_pipeline.py bulk --fp16 --batch-size 48 --grayscale

# Bulk embed with Qwen
python model/run_pipeline.py bulk --model qwen-vl --batch-size 20 --grayscale

# Incremental (new products from last 24h)
python model/run_pipeline.py incremental --fp16 --batch-size 48 --grayscale

# Verify product ID integrity
python model/run_pipeline.py test --fp16 --grayscale

# Visual quality inspection (generates HTML report)
python model/run_pipeline.py inspect

# Side-by-side model comparison
python model/run_pipeline.py compare

# Check coverage stats
python model/run_pipeline.py bulk --verify-only
```

The launcher handles venv switching automatically — you stay in `project-nista-env` and the launcher spawns subprocesses using `venv-marqo` or `venv-qwen` as needed.

### Adding a New Model

1. Create a new adapter class in `model/adapters/` extending `BaseEmbedder`.
2. Implement all abstract methods (`load`, `embed_images_batch`, properties).
3. Register the new key in `EmbedderFactory`.
4. Create a new venv with the model's dependencies.
5. Add new `embedding_vector_<model>` + `embedded_at_<model>` columns to the `products` table.
6. Test with `run_pipeline.py bulk --model <key> --test`.

## Qwen Visual Tagging

Qwen3-VL-Embedding-2B is used for structured tagging, not embedding search. It accepts product image + title + description and outputs tags across multiple categories:

| Category | Example Values |
|----------|---------------|
| Silhouette | oversized, fitted, relaxed, A-line |
| Colour palette | earth tones, pastels, jewel tones, neutrals |
| Aesthetic | minimalist, bohemian, maximalist, vintage, futuristic |
| Occasion | everyday, going out, work, special events, sports |
| Material cues | cotton, linen, silk, knit, denim |

Each tag has a confidence score (0–1). Tags below 0.5 are filtered for display. Output writes to `product_tags` table (product_id, tag_type, tag_value, confidence_score).

## Style Attribute Extraction

Zero-shot classification using FashionSigLIP embeddings to extract structured attributes:
- Waist fit, sleeve length, neckline, hem length
- Fabric texture, colour temperature, pattern type
- Overall aesthetic classification

Output writes to `style_attributes` JSONB column + a 67-dim `style_vector`. See `Backend/attribute-extraction/README.md` for the full taxonomy and extraction pipeline.

## Taste Vector System (Personalisation)

### Seeding During Onboarding

**v0 (5-step onboarding, no swipes):**
1. User selects lifestyle activities (Step 4) → maps to occasion weights.
2. User selects brands (Step 5) → brands' style vectors are averaged to create a **brand-prior vector**.
3. Brand-prior vector × occasion weights = initial per-occasion taste vectors (everyday, going_out, work, special).

**P1 enhancement (swipe mechanic):**
When implemented, Step 6 adds 10 product swipe cards. The initial taste vector becomes: `(0.6 × brand_prior) + (0.4 × swipe_signal)`.

### Session Recalculation

At session start, interactions from the previous session update taste vectors:

```
new_vector = (0.7 × existing_vector) + (0.3 × session_signal)
```

This 70/30 blend keeps the feed fresh while maintaining long-term preference stability. The `session_signal` is computed from all interaction signals in the previous session, weighted by their signal type (see ARCHITECTURE.md Interaction Signals table).

### Per-Occasion Vectors

Each user has four taste vectors:
- `everyday_vector` — default lifestyle
- `going_out_vector` — social, evening
- `work_vector` — professional
- `special_vector` — events, formal

When the user has an occasion filter active on the feed, interactions update the relevant occasion vector. When "All" is selected, interactions update the primary vector (which blends across occasions).

### Cold Start Handling

Without swipes in v0, cold start is more likely. The fallback chain:
1. **Brand-prior seeding** (from onboarding brands selection) — weaker than brand+swipe but sufficient.
2. **Editorial picks** (~20 curated products) — served to users with < 5 interactions.
3. **Trending** (most-saved products in last 7 days) — secondary fallback.
4. **Graduation:** After 5+ meaningful interactions (likes + saves + detail taps), the system computes a real taste vector and switches to the personalised feed.

## Personalised Search

Search doesn't just return relevance-ranked results — it reflects the user's profile:

1. **Pre-filtering by user fingerprint:** Gender expression, preferred sizes, and price range (from onboarding) are applied before search results are ranked. This means a user who selected "Womenswear" and shops in the $50–$200 range sees results filtered to match, without manual filtering.

2. **Taste vector re-ranking:** After Marqo returns relevance-ranked results (product vectors), the results are re-ordered by similarity to the user's taste vector (style vectors). Two users searching the same query see different result orderings.

## Enrichment Pipeline Interface

The enrichment pipeline is triggered by the ingestion layer (Avanti's domain). The interface is simple:

1. Ingestion writes a new product row with `enrichment_status = 'pending'`.
2. The enrichment pipeline picks up pending products.
3. On completion, it writes `product_tags`, `product_embeddings`, and `style_attributes` rows.
4. It sets `enrichment_status = 'completed'`.

The app reads these tables directly — no separate enrichment API needed.

## Evaluation & Quality

### Embedding Quality

Use the built-in evaluation tools:

- `run_pipeline.py inspect` — Generates an HTML report showing nearest-neighbour results for sample products. Visually verify that similar products cluster together.
- `run_pipeline.py compare` — Side-by-side comparison of results across models.
- `scripts/eval/similarity_inspector.py` — More detailed similarity analysis.
- `scripts/eval/model_comparison.py` — Quantitative model comparison.

### What "Good" Looks Like

- **Search:** Querying "navy blazer" returns blazers (not random blue items). Top 10 results are visually coherent.
- **Feed:** A user who likes minimalist earth-tone clothing sees a coherent feed across dresses, tops, and accessories — not just the exact items they've interacted with.
- **Cold start → warm:** Feed quality should noticeably improve after 10–15 interactions.
- **Occasion specificity:** Work feed and Going Out feed for the same user should look meaningfully different.

## Before You Start Coding

1. Read `Backend/model/README.md` for the full pipeline documentation.
2. Check which models are available: `python model/run_pipeline.py list`
3. If working on personalisation, read `Context/ARCHITECTURE.md` Personalisation Layer section.
4. If working on search ranking, read `Context/specs/search.md` for the full spec.
5. Verify GPU is available: `nvidia-smi` (models run on CPU too, but much slower).

## Key Decisions Still Open

- **Marqo hosting:** Currently local GPU only. Cloud deployment (Marqo Cloud or self-hosted VM) needed before the frontend can query vectors in production.
- **Taste vector storage:** Currently in `user_taste_profiles` table. May need to move to a vector index if query performance degrades at scale.
- **Qwen vs alternatives:** Qwen3-VL-2B is the current tagger. Effectiveness still being evaluated. May swap if tag quality is insufficient.
- **Style vector dimensionality:** 67-dim is current. May change as the attribute taxonomy evolves.
