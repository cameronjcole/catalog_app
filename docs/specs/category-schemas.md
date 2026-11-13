# Spec: Category schemas and how they evolve

- **Status:** Draft
- **Date:** 2026-11-12
- **Related:** [Ingestion](ingestion.md) · [Search and answer](search-and-answer.md) ·
  [ADR 0002: storage](../adr/0002-postgres-jsonb-pgvector.md) · [ADR 0003: enrich on save](../adr/0003-enrich-on-save.md)

## Purpose

This spec defines **what we extract from each post** during enrichment, and **how that set grows** as Catalog
supports more kinds of content. The enrichment model's output must match these schemas, and search filters
can only use fields defined here.

## Structure: three layers

```
┌──────────────────────────────────────────────────────────────┐
│ 3. User organization: collections, the user's own tags      │  owned by the user, no schema
├──────────────────────────────────────────────────────────────┤
│ 2. Category blocks: recipe, travel, (restaurant, ...)       │  curated by us, versioned
├──────────────────────────────────────────────────────────────┤
│ 1. Core fields: every post, every category                  │  stable, rarely changes
└──────────────────────────────────────────────────────────────┘
```

The key idea: **search works for any post through the core fields alone**, even one that matches no category.
Category blocks make specific questions sharper, but nothing depends on a category existing. That's what lets
us add categories gradually without breaking anything.

Stored in the `attributes` JSONB column as:

```json
{
  "core": { ... },
  "categories": {
    "recipe": { "schema_version": "1.0", ... },
    "travel": { "schema_version": "1.0", ... }
  }
}
```

A post can have **several category blocks** (a Bangkok street-food video is both `recipe` and `travel`).
`primary_category` (a regular column) is the one the app shows first.

## Global rules for all fields

- **Unknown means `null`**, not a missing key. Lists with nothing in them are `[]`.
- **Ground facts in the post.** The model may normalize and infer hierarchy (Oregon → `US`), but must not
  invent facts (no cook time if the video doesn't give one).
- **Use standard codes where they exist:** ISO 3166-1 alpha-2 for countries (`US`, `CN`, `JP`), ISO 639-1
  for languages.
- **Enums are lowercase, singular, and from a fixed list.** Free text goes in clearly free-text fields only.
- **Locations are core, not travel-only.** Restaurants, events and things to do all have places too.

## Layer 1: Core fields (all posts)

| Field | Type | Notes |
|---|---|---|
| `title` | string | Short, human-readable; generated if the post has none |
| `summary` | string | 2–4 sentences on what the post is about |
| `primary_category` | enum or `other` | One of the defined categories, else `other` |
| `suggested_category` | string \| null | Free-text label when `other`, e.g. `"workout"`. Feeds category discovery (below). |
| `tags` | string[] | 3–10 lowercase topical tags |
| `locations` | Location[] | Every place the post is about (see below); may be empty |
| `people_and_brands` | string[] | Named creators, chefs, brands or venues mentioned |
| `mood` | string[] | Feel or tone, e.g. `cozy`, `adventurous`, `romantic` |
| `language` | string | ISO 639-1 code of the main spoken or written language |
| `content_depth` | enum | `metadata` \| `caption` \| `full`: how much the enrichment had to work with |

**Location**

| Field | Type | Notes |
|---|---|---|
| `name` | string \| null | Specific place: "Crater Lake", "Joe's Pizza" |
| `city` | string \| null | |
| `region` | string \| null | State, province or prefecture |
| `country` | string \| null | ISO 3166-1 alpha-2; inferred from region or city when unambiguous |
| `kind` | enum \| null | `city` \| `natural_site` \| `landmark` \| `venue` \| `region` \| `country` |

(Coordinates via geocoding are a likely later addition for map and "near me" features.)

## Layer 2: Category blocks

### `recipe` (v1.0)

| Field | Type | Notes |
|---|---|---|
| `dish_name` | string \| null | |
| `cuisine` | string[] | e.g. `italian`, `thai`, `mexican` |
| `meal_type` | enum[] | `breakfast` \| `lunch` \| `dinner` \| `snack` \| `dessert` \| `drink` \| `side` |
| `main_ingredients` | string[] | The 3–6 defining ingredients, normalized (`salmon`, not `2 salmon fillets`) |
| `ingredients` | {name, quantity}[] | Full list when shown; `quantity` is free text or `null` |
| `dietary` | enum[] | `vegetarian` \| `vegan` \| `gluten_free` \| `dairy_free` \| `high_protein` \| ... (only when evident) |
| `flavor_profile` | string[] | e.g. `spicy`, `citrusy`, `savory`, `sweet` |
| `season` | enum[] | `spring` \| `summer` \| `fall` \| `winter`: when it's best suited |
| `effort` | enum \| null | `easy` \| `medium` \| `involved` |
| `total_time_minutes` | integer \| null | Only when stated or clearly shown |
| `equipment` | string[] | e.g. `air fryer`, `dutch oven` |

### `travel` (v1.0)

Places live in core `locations`. This block describes the trip or experience.

| Field | Type | Notes |
|---|---|---|
| `experience_type` | enum[] | `hiking` \| `beach` \| `city` \| `food` \| `culture` \| `nightlife` \| `road_trip` \| `nature` \| ... |
| `activities` | string[] | Specific things to do |
| `best_season` | enum[] | `spring` \| `summer` \| `fall` \| `winter` |
| `activity_level` | enum \| null | `relaxed` \| `moderate` \| `strenuous` |
| `budget_level` | enum \| null | `budget` \| `mid` \| `luxury` |
| `suggested_duration_days` | integer \| null | Only when stated |
| `travel_style` | string[] | e.g. `solo`, `family`, `couples`, `backpacking` |
| `tips` | string[] | Practical tips the post gives |

### Candidate categories (not yet defined)

Listed roughly by expected demand. Each needs its own schema section before it ships (see process below).

| Candidate | Notes |
|---|---|
| `restaurant` | Places to eat or drink. Heavily uses core `locations`; adds cuisine, price, dishes to try, reservation needs. Likely the first addition. |
| `activity` | Things to do (in your city or elsewhere): events, classes, attractions. Overlaps with travel; the difference is a single activity vs. a trip. |
| `product` | Things to buy: fashion, gadgets, home goods. Brand, price range, where to buy. |
| `workout` | Exercises and routines: body area, equipment, duration, difficulty. |
| `media` | Books, shows, films, games, music: title, creator, genre, platform. |
| `art_inspiration` | Art, design, style and decor ideas: medium, style, colors. Mostly tags and mood; may need few fields. |
| `diy_home` | Projects and how-tos: materials, tools, difficulty. |

## Evolving categories

### What we decided not to do

| Option | Why not (for now) |
|---|---|
| **A fixed, closed list** | Content is too varied; posts outside the list would get worse results forever. |
| **Fully self-service schemas** (users define their own categories and fields) | Every user's posts would need custom extraction, filters and prompts. That's hard to make reliable and hard to search across, and it's a lot to ask of users. |
| **Fully automatic categories** (the model invents categories and fields on the fly) | Inconsistent naming ("food", "recipes", "cooking") and unstable fields make filtering unreliable. |

### What we will do: curated categories, discovered from real use, with user-owned organization

1. **We own the category schemas (layer 2).** Categories and their fields are designed, versioned and
   tested by us, in this document. That keeps extraction consistent and filters reliable.
2. **The core layer covers everything else.** Posts that match no category still get a summary, tags,
   locations and mood, and they are fully searchable. `other` is a supported state, not a failure.
3. **Real usage tells us what to add.** Every `other` post records a `suggested_category`. We regularly
   review those labels (counts, clusters, sample posts) to decide what to define next. The data decides,
   not guesswork.
4. **Users organize their own way (layer 3), without schemas.** People group things differently ("Tokyo
   trip", "date night ideas", "things to do in Portland"). That's what **collections** and the user's own
   **tags** are for. They are user-owned labels and don't change extraction. Collections are their own
   feature spec (to be written).
5. **Later, maybe: per-user custom fields.** If users clearly want to extract things we don't support
   (e.g. "wine pairing" on every recipe), we could offer custom fields extracted by a prompt, applied only to
   that user's posts. Not for v1; revisit with real data.

### Process for adding or changing a category

1. **Propose:** open a short proposal with usage evidence (`suggested_category` counts and sample posts),
   and draft the schema section in this document.
2. **Define:** finalize the fields using the global rules. Prefer reusing core fields over adding category
   fields. Add the schema to the enrichment code and prompt.
3. **Evaluate:** run enrichment on a sample of real posts (both matching and not) and check accuracy and
   whether it confuses existing categories.
4. **Ship:** deploy. New saves get the new block.
5. **Backfill:** re-run enrichment (from stored extractions, see [ingestion](ingestion.md#reprocessing)) on
   existing posts likely to match: those with `suggested_category` near the new category, and posts in
   related categories.
6. **Enable search:** add the new fields to the query-understanding context so they can be used as filters.

### Versioning rules

Every category block stores `schema_version` (`major.minor`).

| Change | Version bump | Backfill required? |
|---|---|---|
| Add an optional field or enum value | Minor (1.0 → 1.1) | Optional: old posts have `null` until re-enriched |
| Rename or remove a field, change a type, change an enum's meaning | Major (1.x → 2.0) | Yes: search must not mix the old and new meaning |
| Add a new category | New block at 1.0 | Targeted backfill (step 5 above) |

Guardrails:
- **Prefer fields over categories.** Add a category only when the questions people ask clearly differ
  from existing ones.
- **Keep the top level small**, around a dozen categories. Use subtypes inside a category (e.g.
  `experience_type`) before adding near-duplicate categories.
- **Never break the core layer** without a major, fully backfilled migration.

## Open questions

1. **Restaurant in v1?** It's high value and mostly core fields plus a few more.
2. **Food naming:** is "food" one category with recipe and restaurant subtypes, or two separate categories?
   (The current draft has two.)
3. **Confidence:** do we store a per-field or per-block confidence so search can down-weight guesses?
4. **Category review cadence and tooling:** a simple internal dashboard over `suggested_category`?
5. **Where the canonical schemas live:** JSON Schema files in the server code, generated from or checked
   against this document?
