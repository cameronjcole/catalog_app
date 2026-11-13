# 0002: Postgres with JSONB and pgvector as the single data store

- **Status:** Accepted
- **Date:** 2026-11-12

## Context

Catalog stores saved posts, plus what we learn about them: summaries, category-specific details
(ingredients for a recipe, locations for a travel spot), keyword-searchable text and embeddings.

Three kinds of query have to work together:

1. **Exact filters:** "travel posts in China", "recipes with salmon".
2. **Keyword search:** names, hashtags and rare words that embeddings handle poorly.
3. **Semantic search:** "light summery dinners", "relaxing places to visit".

The details differ by category and will change as we add categories
(see [category schemas](../specs/category-schemas.md)), so a fixed set of columns won't fit.

Options considered:

- **Postgres with JSONB and pgvector:** one database for relational data, flexible documents and vectors.
- **Postgres plus a dedicated vector database** (Pinecone, Qdrant, etc.): two systems to keep in sync.
- **A search engine as the store** (Solr, Elasticsearch, OpenSearch): strong ranking, but users, accounts
  and jobs would still need a separate primary database.
- **A document database** (MongoDB, DynamoDB): flexible shape, but weaker at joins, transactions and
  mixed filter-plus-vector queries.

## Decision

Use **one Postgres database**, with each post stored as a hybrid row:

| Kind of data | Stored as | Example |
|---|---|---|
| Fields every post has | Regular columns | `id`, `user_id`, `url`, `platform`, `primary_category`, `status`, `saved_at` |
| Category-specific details | A `jsonb` column with a GIN index | `{"travel": {"best_season": ["summer"], ...}}` |
| Keyword search | A `tsvector` column with a GIN index | Built from the summary, caption, transcript and tags |
| Semantic search | A `vector` column (pgvector) with an HNSW index | The embedding of the post's search document |

Rules that come with this:

- **The JSON is flexible in storage but strict in shape.** Every category has a versioned schema, and the
  enrichment step must produce output that matches it. Unknown values are `null`, not missing keys.
- **Fields get promoted when they earn it.** A JSONB field that becomes central to querying moves into its
  own column or table, for example `post_locations` once one-post-to-many-places queries matter.
- **Every query is scoped to `user_id`.** Row-level security is likely; to be confirmed when the server is built.

Illustrative only. The real schema is defined when the server is built:

```sql
CREATE TABLE posts (
  id               uuid PRIMARY KEY,
  user_id          uuid NOT NULL,
  url              text,
  platform         text NOT NULL,
  status           text NOT NULL,     -- pending | processing | ready | failed
  primary_category text,
  summary          text,
  attributes       jsonb,             -- core fields + per-category blocks
  search_text      tsvector,
  embedding        vector(1024),      -- size depends on the embedding model
  saved_at         timestamptz NOT NULL
);
```

## Consequences

- **Pros:** one system to run, back up and secure. Filters, keyword search and vector search fit in a single
  SQL query. Real transactions across users, posts and jobs. Every major cloud offers managed Postgres
  with pgvector.
- **Cons:** built-in Postgres full-text ranking is not true BM25. If ranking quality falls short, add a BM25
  extension (for example ParadeDB `pg_search`), as long as our host supports it. At very large scale a
  dedicated search engine may beat Postgres, but we're nowhere near that.
- **Hosting:** whatever host we pick must support the `vector` extension (and a BM25 extension, if we adopt one).
- **Revisit if:** search quality or latency can't be fixed with indexes and extensions, or vector volume
  grows far beyond what a single Postgres instance handles comfortably.
