# Spec: Ingestion (saving a post)

- **Status:** Draft
- **Date:** 2026-11-12
- **Related:** [ADR 0002: storage](../adr/0002-postgres-jsonb-pgvector.md) ·
  [ADR 0003: enrich on save](../adr/0003-enrich-on-save.md) ·
  [Category schemas](category-schemas.md) · [Search and answer](search-and-answer.md)

## Problem

People save posts inside Instagram, TikTok and other apps, then can never find them again. Saving into
Catalog has to take one tap from inside those apps. After that, Catalog does the work of understanding
the post so it can be searched later.

## Goals

- Saving from any supported app takes **one share action and under 2 seconds** before the user is back in their app.
- A saved post is **never lost**, even offline or when processing fails.
- Each saved post becomes a **searchable record**: core fields, category details, keyword index and embedding.
- We can **reprocess** posts when enrichment improves, without the user doing anything.

## Non-goals (for this spec)

- Searching and answering questions ([search and answer](search-and-answer.md)).
- Bulk import of an existing saved-posts history (a likely later feature).
- Android or web clients.

## Components

| Component | Role |
|---|---|
| **Source app** | Instagram, TikTok, YouTube, X, Photos, Safari, ... |
| **Share Extension** | iOS extension that receives what the source app shares and hands it to Catalog |
| **Catalog app** | Shows saved posts and their processing status. Shares an App Group with the extension. |
| **API server** | Accepts saves, authenticates, removes duplicates, records the post, queues processing |
| **Job queue and worker** | Runs the slow steps in the background with retries |
| **Object storage** | Holds uploaded media and extracted frames (S3, GCS, etc.) |
| **Postgres** | The source of truth for posts, extractions and enrichment ([ADR 0002](../adr/0002-postgres-jsonb-pgvector.md)) |
| **Model providers** | Speech-to-text, multimodal enrichment model, embedding model |

## End-to-end flow

```
Source app ──share──▶ Share Extension ──▶ API: create post ──▶ queue job ──▶ (user is done)
                                                                   │
            ┌──────────────────────────────────────────────────────┘
            ▼
   1. Acquire  ──▶  2. Extract  ──▶  3. Enrich  ──▶  4. Index  ──▶  ready
   (metadata,       (transcript,     (model →        (search doc,
    media)           frames, OCR,     core fields +   tsvector,
                     caption)         category data)  embedding)
```

### Step 0: Share (on device)

1. The user taps **Share → Catalog** in the source app.
2. The Share Extension reads what it was given. Depending on the app, this is one or more of:
   - **A URL**, the most common case for Instagram and TikTok
   - **A video file** (usually when shared from Photos)
   - **One or more images**
   - **Text**: a caption, or a URL embedded in text
3. The extension shows a small confirmation sheet: a preview, an optional **note** field, and Save.
   We may skip the sheet in favor of instant save; see open questions.
4. On Save, the extension:
   - writes the save to a **local outbox** in the App Group container, so nothing is lost if offline, then
   - calls the API (see Step 1) using the user's token, shared through the App Group keychain.
5. The extension closes. The user is back in the source app.

> **To validate first:** what each source app actually shares. Build a spike Share Extension that logs
> the payload received from Instagram, TikTok, YouTube, X, Photos and Safari, and record the results here.

### Step 1: Create the post (API, synchronous)

1. The server authenticates the request.
2. It **normalizes the URL**: removes tracking parameters, resolves short links, and detects the platform.
3. It **removes duplicates**: if this user already saved this normalized URL, it returns the existing post
   (with the new note appended, if one was given).
4. If media was included, the client uploads it straight to object storage using a short-lived upload URL
   the server issues, rather than sending it through the API.
5. It creates the `posts` row with `status = pending` and queues an ingestion job.
6. It responds right away with the post (`201`). The extension removes it from the outbox.

### Step 2: Acquire (worker)

Get as much of the original content as we're able to. **The approach is undecided**; see open questions, and
this needs its own ADR. Candidate sources, from least to most content:

- Open Graph / oEmbed metadata: title, author, thumbnail, sometimes the caption
- Media the user shared directly (video or images from Step 0)
- Anything else we decide is acceptable under each platform's terms

Where we only get metadata, the post still moves forward. It just has less to work with.

### Step 3: Extract (worker)

Turn the raw content into text and representative images:

| Input | Output |
|---|---|
| Video audio | **Transcript** (speech-to-text), with language detected |
| Video frames | **Keyframes**: a small, fixed number sampled across the video, plus scene changes |
| Frames and images | **On-screen text** (OCR): recipe cards, place names, prices |
| Caption, hashtags, author | Stored as given |
| User's note | Stored as given, and weighted highly; it's the user's own words |

Everything here is saved as a **raw extraction** record tied to the post, so later reprocessing never needs
the source media again.

### Step 4: Enrich (worker)

One structured-output call to a multimodal model, with the transcript, keyframes, OCR text, caption and note
as input. Output is defined in [category schemas](category-schemas.md):

- **Core fields** every post gets: title, summary, primary category, tags, locations, and so on.
- **Category blocks** for every category that applies. A street-food video in Bangkok can carry both `recipe`
  and `travel` blocks.
- `enrichment_version` and each block's `schema_version` are recorded.

The model must stay grounded in the input. It may **normalize and infer hierarchy** ("Crater Lake, Oregon" →
country `US`, region `Oregon`), but it must not invent facts the post doesn't support. Unknown values are `null`.

### Step 5: Index (worker)

1. Build the **search document**: a text rendering of the title, summary, core fields, category blocks,
   caption and note. It's the same text a model sees when answering questions.
2. Compute the keyword index (`tsvector`) from the search document plus the transcript.
3. Compute the **embedding** of the search document.
4. Set `status = ready`. The app finds out by refreshing, or by push notification if we add one later.

## Post status

```
pending ──▶ processing ──▶ ready
                │
                └──▶ failed  (retry: automatic with backoff, then manual from the app)
```

- A post with only metadata (Step 2 got little) still reaches `ready`. We record what we had to work with
  (`content_depth: metadata | caption | full`) so the app can show it and suggest adding a note.
- `failed` posts stay visible in the app with a retry action. Saves are never silently dropped.

## Data produced

| Record | Contents |
|---|---|
| `posts` | Core columns, `attributes` (JSONB), `search_text`, `embedding`, `status` |
| `post_media` | References to uploaded media and extracted keyframes in object storage |
| `post_extractions` | Transcript, OCR text, caption, metadata; versioned |
| (later) `post_locations` | One row per post per place, if location queries need it ([ADR 0002](../adr/0002-postgres-jsonb-pgvector.md)) |

## Reprocessing

When enrichment changes (a new category, new fields, a better prompt, a new model), a backfill job re-runs
**Steps 4–5** from the stored `post_extractions` for affected posts. A post's `enrichment_version` and block
`schema_version` values tell us which posts are out of date. See
[category schemas → evolution](category-schemas.md#evolving-categories).

## Deleting

When a user deletes a post, its row, extractions, media and embeddings are all removed. No stale copies are
left in object storage or derived indexes.

## Open questions

1. **Content acquisition (the biggest one):** link-only metadata, user-supplied files, or a hybrid? What is
   acceptable under each platform's terms? → ADR 0004.
2. **Confirmation sheet or instant save?** Instant is faster; the sheet allows a note and a preview.
3. **Media retention:** keep original media after extraction, or delete it to cut cost and privacy exposure?
4. **Limits:** maximum video length or size, keyframes per video, and handling of carousels with many images.
5. **Sharing work across users:** if two users save the same public post, can we reuse its extraction?
   This saves cost but has privacy implications.
6. **Model choices:** which providers for transcription, enrichment and embeddings, and the target cost per save.
7. **Notifications:** is polling on app open enough, or do we want a push when processing finishes?
