# Spec: Search and answer

- **Status:** Draft
- **Date:** 2026-11-12
- **Related:** [Ingestion](ingestion.md) · [Category schemas](category-schemas.md) ·
  [ADR 0002: storage](../adr/0002-postgres-jsonb-pgvector.md)

## Problem

Once posts are saved and enriched, users need to get them back by asking in their own words, whether they
want a list of matching posts or an answer drawn from them.

## Goals

- Users can type a natural question and get back **the right posts**, or **an answer built from their posts**.
- Every answer **shows its sources**: the posts it came from, with platform and thumbnail.
- "List all" questions return **everything that matches**, not just the top few.
- Answers use **only the user's own saved posts**. No made-up posts or facts.

## Non-goals (for this spec)

- Multi-turn chat with follow-up questions (likely a later version).
- Searching across other users' posts or public content.
- Recommending new content the user hasn't saved.

## Kinds of question

| Kind | Example | What the user wants | Answer path |
|---|---|---|---|
| **Find** | "show me my salmon recipes" | The matching posts | Retrieve → list of post cards |
| **List or filter** | "places I saved in China" | *All* matches for a constraint | Filter (no top-k cap) → paginated cards |
| **Answer** | "what summer recipes do I have that are quick?" | A written answer drawn from posts | Retrieve → generate → answer + sources |

Find and list questions **skip generation**: they're faster, cheaper, and often exactly what the user wants.

## End-to-end flow

```
query ──▶ 1. Understand ──▶ 2. Retrieve ──▶ 3a. Return post cards           (find / list)
          (cheap model)     (filters +       3b. Generate answer + citations (answer)
                             hybrid search)        │
                                                   ▼
                                            4. App shows answer + source cards
```

### Step 1: Understand the query

A fast, cheap model turns the question into a structured **query plan** (structured output):

```json
{
  "intent": "answer",
  "categories": ["recipe"],
  "filters": { "recipe.season": ["summer"], "recipe.total_time_minutes": { "lte": 30 } },
  "semantic_query": "quick light summer dishes",
  "keywords": ["summer"]
}
```

- Filters can only use fields that exist in the [category schemas](category-schemas.md). The schemas are
  given to the model as context.
- When unsure, prefer **looser** filters: a missed post is worse than an extra one that gets ranked lower.
- For very simple queries (one or two words), we might skip the model and go straight to hybrid search.

### Step 2: Retrieve

Every query is scoped to the user (`user_id`).

1. **Filter:** apply category and structured filters as SQL on columns and JSONB.
2. **Rank:**
   - **List or filter intent:** return *all* filtered posts, sorted by date or relevance, paginated. No top-k cap.
   - **Find or answer intent:** run **hybrid search** on the filtered set:
     - vector similarity of `semantic_query` against post embeddings
     - keyword search of `keywords` (and the raw query) against `search_text`
     - merge the two ranked lists with **reciprocal rank fusion** (RRF)
3. **Group by post** (one result per post) and keep the top **N** (starting point N ≈ 10, to be tuned).

### Step 3a: Return post cards (find, list)

Return the posts with what a card needs: title, thumbnail, platform, primary category, and a short summary.

### Step 3b: Generate an answer (answer intent)

A model receives:
- the user's question
- the **search documents** of the top N posts (see [ingestion](ingestion.md#step-5-index-worker)), each tagged with its post ID

Rules for the model:
- Answer **only** from the given posts. If none fit, say so plainly.
- **Cite** the post IDs used. The server checks that every cited ID was in the retrieved set and drops any that weren't.
- Stream the response so text appears quickly.

### Step 4: Show it in the app

- The answer text, with the **source posts** shown as cards underneath (or inline): platform icon,
  thumbnail, title, and a tap-through to the original post.
- For find and list results, the cards alone.

## Quality and evaluation

- Build a small **eval set** early: about 50 real queries over a sample library, with the expected posts for each.
  It measures retrieval (did the right posts come back?) separately from answer quality.
- Use it to tune N, the RRF weights, the query-understanding prompt, and whether we need BM25.

## Open questions

1. **Keyword ranking:** is built-in Postgres full-text search good enough, or do we need a BM25 extension?
   ([ADR 0002](../adr/0002-postgres-jsonb-pgvector.md))
2. **Follow-up questions:** in v1, or later?
3. **Where search lives in the app:** one search bar that picks the intent automatically, or separate
   "search" and "ask" modes?
4. **N and context size:** how many posts to give the answering model, and how much of each.
5. **Latency targets:** for example, cards in under 1 s and the first answer token in under 2 s.
6. **Model choices** for query understanding and answering, and the cost per question.
