---
name: Generate a typed client from a Hey API registry specification
description: >-
  Resolve an OpenAPI document out of the Hey API registry — latest, by branch,
  by tag, by version, or pinned to an exact commit — and generate a
  production-grade TypeScript or Python client from it.
api: openapi/hey-api-platform-openapi.json
base_url: https://api.heyapi.dev
operations:
  - GET /v1/get/{organization_slug}/{project_slug}
generated: '2026-08-06'
method: generated
source: >-
  Grounded in openapi/hey-api-platform-openapi.json (harvested live),
  https://heyapi.dev/docs/openapi/typescript/integrations and
  https://heyapi.dev/docs/openapi/typescript/get-started. Query parameters are
  verbatim from the spec.
---

# Generate a client from the Hey API registry

## When to use this

You consume an API whose team publishes its OpenAPI to Hey API. Instead of
committing a stale copy of `openapi.json`, you point your codegen at the
registry and get a reproducible build.

## The resolution chain

`<organization>/<project>` shorthand
→ `https://get.heyapi.dev/<organization>/<project>`
→ `308` redirect to `https://api.heyapi.dev/v1/get/<organization>/<project>`

That last URL is the real operation:
`GET /v1/get/{organization_slug}/{project_slug}`.

## Steps

### 1. Authenticate (unless the project is public)

Projects are private by default. Use a **personal API key** from
`app.heyapi.dev/settings/user/apis` for local development, a **project key** in
CI.

Two accepted forms:

- `Authorization: Bearer <api_key>` — preferred
- `?api_key=<api_key>` — for codegens that only take a URL

The query form is what lands the key in proxy logs and shell history. Read it
from an environment variable, never hardcode it:

```ts
input: {
  path: `https://get.heyapi.dev/acme/backend?api_key=${process.env.HEY_API_USER_TOKEN}`,
}
```

The spec also declares an empty security requirement `{}` on this operation, so
public projects — like Hey API's own `hey-api/backend` — resolve anonymously.
Use that one to try the flow with no account.

### 2. Choose which specification you get

Default behaviour returns the **last uploaded** document. Narrow it with the
query parameters the spec declares:

| Parameter | Effect |
|---|---|
| `branch` | Last build from that branch, e.g. `?branch=production` |
| `commit_sha` | An exact document — always returns the same file |
| `tags` | First match among comma-separated custom tags |
| `version` | Last upload matching the OpenAPI `info.version` |
| `latest` | Force the latest |
| `inline` | Content disposition of the response (default `false`) |

**For a reproducible build, pin `commit_sha`.** Anything else can move under
you between CI runs.

### 3. Generate

```sh
# TypeScript — registry shorthand
npx @hey-api/openapi-ts -i acme/backend -o src/client
```

Or in `openapi-ts.config.ts`:

```ts
import { defineConfig } from '@hey-api/openapi-ts';

export default defineConfig({
  input: 'acme/backend',
  output: 'src/client',
  plugins: ['@hey-api/client-fetch', '@hey-api/typescript', '@hey-api/sdk'],
});
```

Any other codegen works too — the endpoint just returns an OpenAPI document:

```sh
npx openapi-typescript https://get.heyapi.dev/acme/backend -o schema.ts
npx orval --input https://get.heyapi.dev/acme/backend --output ./src/client.ts
```

### 4. Regenerate when the spec changes

Register a webhook on the project
(`POST /v1/organizations/{organization_slug}/projects/{project_slug}/webhooks`)
and react to `specification.created`. See
`asyncapi/hey-api-platform-webhooks.yml`.

## Rules that will bite you

- **Pin the generator.** `@hey-api/openapi-ts` is in semver initial development
  (0.x); minors break. The docs install with `-E` for exactly this reason, and
  migration notes are published per breaking release.
- **Node.js 22+ is required.**
- **A `404` here is ambiguous** — it means "no such org/project pair" *or* "you
  cannot see it". A private project fetched anonymously surfaces as `404`, not
  `401`.
- **Self-signed dev certs** need `NODE_TLS_REJECT_UNAUTHORIZED=0`.

## Errors

`{ "error": { "message", "request_id", "status", "timestamp" } }` on `401`,
`403` and `404`. See `errors/hey-api-problem-types.yml`.
