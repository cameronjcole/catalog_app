# Docs

- `specs/`: feature and product specs, one file per feature. We work feature by feature.
- `adr/`: Architecture Decision Records. Numbered and never deleted.
  If a decision changes, write a new ADR that supersedes the old one.

## Where to start

Read in this order:

1. [Ingestion](specs/ingestion.md): how a post gets from a social app into Catalog's database.
2. [Category schemas](specs/category-schemas.md): what we extract from each post, and how categories evolve.
3. [Search and answer](specs/search-and-answer.md): how users find posts and get answers from them.

## Specs

| Spec | Status | Summary |
|---|---|---|
| [Ingestion](specs/ingestion.md) | Draft | Share → save → extract → enrich → index |
| [Category schemas](specs/category-schemas.md) | Draft | Core fields, recipe and travel blocks, the evolution process |
| [Search and answer](specs/search-and-answer.md) | Draft | Query understanding, hybrid retrieval, answers with sources |

## Decisions

| ADR | Status | Decision |
|---|---|---|
| [0001](adr/0001-monorepo-layout.md) | Accepted | Monorepo with `api/`, `docs/`, `ios/`, `server/` |
| [0002](adr/0002-postgres-jsonb-pgvector.md) | Accepted | One Postgres database: columns, JSONB and pgvector |
| [0003](adr/0003-enrich-on-save.md) | Accepted | Enrich posts into structured text on save; embed the text, not the media |
| 0004 (planned) | n/a | Content acquisition: how we get post content from a shared link |
