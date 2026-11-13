# Catalog

Save posts from Instagram, TikTok, and other social platforms, then search them with AI.

## Repository layout

| Path      | What lives here                                                                 |
|-----------|---------------------------------------------------------------------------------|
| `api/`    | OpenAPI spec: the contract between the server and every client. Edit this first. |
| `docs/`   | Product specs (`docs/specs/`) and architecture decision records (`docs/adr/`).    |
| `server/` | Backend service (language and hosting not chosen yet).                           |
| `ios/`    | SwiftUI iOS app, plus a Share Extension for saving posts from other apps.        |

Planned later: `android/`, `web/`, `infra/` (infrastructure-as-code).

## How the pieces fit

```
          api/openapi.yaml
         /       |        \
   server/     ios/     (android/, web/ later)
 implements   generated client code
```

1. Change the API in `api/openapi.yaml`.
2. Update the server to implement it.
3. Regenerate client code in each app.

## Conventions

- Each top-level folder owns its own tooling and build setup. There is no root build system.
- Secrets never get committed. Use `.env` files (gitignored) and commit a `.env.example`.
- Decisions that matter go in `docs/adr/` as short, numbered markdown files.
