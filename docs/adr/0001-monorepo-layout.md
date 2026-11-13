# 0001: Monorepo layout

- **Status:** Accepted
- **Date:** 2026-11-12

## Context

Catalog starts as a SwiftUI iOS app with a separate, non-Swift backend. Android and web clients may follow.

## Decision

Use one repository with a top-level folder per deliverable (`ios/`, `server/`, later `android/` and `web/`),
plus `docs/` for specs and `api/` for an OpenAPI spec that every client and the server build against.

## Consequences

- An API change and its client changes can land in a single commit.
- Each folder keeps its own tooling. No monorepo build tool until there are shared TypeScript packages.
- CI workflows should use path filters so each app builds only when its folder changes.
