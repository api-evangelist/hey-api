---
name: Publish an OpenAPI specification to the Hey API registry from CI
description: >-
  Upload an OpenAPI document to a Hey API Platform project on every build, with
  the CI provenance attached, so downstream clients can regenerate against a
  pinned commit.
api: openapi/hey-api-platform-openapi.json
base_url: https://api.heyapi.dev
operations:
  - POST /v1/specifications
generated: '2026-08-06'
method: generated
source: >-
  Grounded in openapi/hey-api-platform-openapi.json (harvested live) and
  https://heyapi.dev/docs/openapi/typescript/integrations. The spec declares no
  operationIds; operations are addressed by METHOD + PATH, which are verbatim
  from the spec. The overlay names (uploadSpecification) are API Evangelist
  annotations, not provider identifiers.
---

# Publish an OpenAPI specification to Hey API

## When to use this

You own an API and you want its OpenAPI document to be the single source every
downstream SDK generates from. Hey API stores each upload as a versioned,
build-attested record, so a consumer can pin to a branch, a tag, a semantic
version, or an exact commit.

## Credentials

Use a **project API key**, not a personal key. Personal keys cannot upload and
will fail with `403`.

- Create it at `app.heyapi.dev` → your project → **Integrations** → **APIs**.
- Send it as `Authorization: Bearer <api_key>`.

## Steps

### 1. Upload the document

`POST /v1/specifications`

Content type is `multipart/form-data`. The only required part is
`specification`, the OpenAPI file itself (`application/json` or
`application/yaml`).

Every other part is CI provenance, and you should send as much of it as your
runner exposes — it is what makes the record addressable later:

| Part | Example |
|---|---|
| `specification` | the file (required) |
| `repository` | `hey-api/platform` |
| `branch` | `feat/cool-feature` |
| `branch_base` | `main` |
| `default_branch` | `main` |
| `commit_sha` | `678e8b7a0d1f495c4a6d011fb76c136955c7f260` |
| `ci_platform` | `github` |
| `event_name` | `pull_request` |
| `workflow` / `job` | `CI` / `ci` |
| `run_id` / `run_number` | `13432046322` / `1` |
| `ref` / `ref_type` | `refs/pull/1/merge` / `branch` |
| `actor` / `actor_id` | the user who triggered the build |
| `tags` | comma-separated, e.g. `dev` |
| `dry_run` | set it to validate without persisting |

On success you get `200` with a `Specification` object: `id`, `project_id`,
`api_key_id`, `created_at`, plus everything you sent.

### 2. Prefer the first-party GitHub Action

If you are on GitHub Actions, do not hand-roll the multipart call — the action
collects the provenance for you:

```yaml
- name: Upload OpenAPI spec
  uses: hey-api/upload-openapi-spec@v1.3.0
  with:
    path-to-file: path/to/openapi.json
    tags: optional,custom,tags
  env:
    API_KEY: ${{ secrets.HEY_API_TOKEN }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 3. Verify it landed

`GET /v1/organizations/{organization_slug}/projects/{project_slug}/specifications`

Cursor paginated — `after`, `before`, `limit` (default 10, max 100). The
response is `{ "items": [...], "filters": { start_cursor, end_cursor,
has_next_page, has_previous_page } }`.

## Rules that will bite you

- **Not idempotent.** Re-running the upload creates another `Specification`
  record. There is no `Idempotency-Key`. Guard the step in your workflow rather
  than relying on the API to dedupe.
- **Dry-run first on a new pipeline.** Send `dry_run` to exercise the whole
  path — credential, encoding, provenance — without writing a record.
- **`413 Content Too Large` is real and undocumented.** No maximum size is
  published. If a large bundled spec fails, that is the wall you hit.
- **No `400` is declared anywhere in the contract**, including for a malformed
  OpenAPI document. Validate before you upload.
- **No rate-limit contract.** No `429`, no `RateLimit-*` headers. Do not assume
  a retry budget.

## Errors

All failures return `{ "error": { "message", "request_id", "status",
"timestamp" } }` — plain `application/json`, not RFC 9457. Quote `request_id`
when you report a problem.

| Status | Cause |
|---|---|
| `401` | Missing/invalid key |
| `403` | Wrong key kind (personal instead of project), or no access to the project |
| `413` | The document is too large |

See `errors/hey-api-problem-types.yml`.
