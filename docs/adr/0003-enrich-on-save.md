# 0003: Enrich posts into structured text on save, and embed the text, not the media

- **Status:** Accepted
- **Date:** 2026-11-12

## Context

Saved posts are mostly short videos and images. To search them we need something we can embed and filter on.
We could embed the raw media directly with a multimodal embedding model, or first turn each post into text.

## Decision

When a post is saved, a background job:

1. **Extracts** the raw material: transcript, sampled frames, on-screen text, caption and hashtags.
2. **Enriches** it with a multimodal model, producing **structured fields plus a written summary** that
   match the [category schemas](../specs/category-schemas.md).
3. **Indexes** the result: builds a search document from the enriched text, embeds it, and stores the
   keyword index.

We embed the **generated text**, not the raw video or images.

We keep the raw extraction and record which enrichment version produced each result, so posts can be
re-enriched later without fetching the media again.

## Consequences

- **Pros:** the stored data is readable and debuggable, and can be shown in the app. Structured fields make
  exact filters possible ("country = CN"). Switching embedding models means re-embedding text, not
  reprocessing video. The same text feeds the answering model.
- **Cons:** each save costs model calls (transcription, enrichment, embedding), so cost per save needs
  watching. Anything the enrichment step misses can't be found by search until we re-enrich.
- **Revisit if:** multimodal embeddings clearly beat text embeddings on our own eval set. Even then they
  would be added alongside the text, not replace it.
