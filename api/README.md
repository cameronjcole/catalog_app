# API

`openapi.yaml` is the source of truth for the Catalog HTTP API (OpenAPI 3.1).

Workflow: change the spec here first, then update the server, then regenerate client code in each app.

Validate it locally:

```bash
npx @redocly/cli lint api/openapi.yaml
```
