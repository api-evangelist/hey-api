---
name: Provision a Hey API organization, project, keys and webhooks
description: >-
  Stand up the tenancy an API team needs on the Hey API Platform — organization,
  project, members, API keys and event webhooks — and rotate credentials safely.
api: openapi/hey-api-platform-openapi.json
base_url: https://api.heyapi.dev
operations:
  - GET /v1/users/me
  - POST /v1/organizations
  - POST /v1/organizations/{organization_slug}/members
  - POST /v1/organizations/{organization_slug}/projects
  - POST /v1/organizations/{organization_slug}/projects/{project_slug}/api-keys
  - DELETE /v1/organizations/{organization_slug}/projects/{project_slug}/api-keys/{api_key_id}
  - POST /v1/organizations/{organization_slug}/projects/{project_slug}/webhooks
generated: '2026-08-06'
method: generated
source: >-
  Grounded in openapi/hey-api-platform-openapi.json (harvested live from
  https://api.heyapi.dev/v1/get/hey-api/backend). Operations are addressed by
  METHOD + PATH verbatim from the spec, which declares no operationIds.
---

# Provision Hey API tenancy

## Credential for this whole flow

Every operation below is secured by **Clerk** — a session JWT, not an API key.
`Authorization: Bearer <clerk_jwt>`. API keys only work on
`POST /v1/specifications` and the spec-download endpoint. If you try to drive
management endpoints with an API key you get `401`.

## Steps

### 1. Identify yourself

`GET /v1/users/me` → a `User` with `id`, `email`, `clerk_user_id` and `roles[]`.
Keep the `id`; the personal-key and waitlist paths are keyed on it.

`GET /v1/users/{user_id}/roles` returns each membership with a `status` of
`active`, `invited` or `suspended`. Only `active` gets you through the
organization endpoints.

### 2. Create the organization

`POST /v1/organizations` → `Organization` with `name` and `slug`.

`GET /v1/organizations` lists them, cursor paginated (`after`, `before`,
`limit`; default 10, max 100).

### 3. Add members

- `POST /v1/organizations/{organization_slug}/members`
- `GET /v1/organizations/{organization_slug}/members`
- `DELETE /v1/organizations/{organization_slug}/members/{user_id}`

### 4. Create the project

`POST /v1/organizations/{organization_slug}/projects` → `Project` with `name`,
`slug`, `default_branch`.

**Name it to match your GitHub structure.** The `organization_slug/project_slug`
pair is public API surface — it becomes the codegen input every consumer types
(`npx @hey-api/openapi-ts -i acme/backend`). Renaming it later breaks their
builds.

### 5. Issue a project API key

`POST /v1/organizations/{organization_slug}/projects/{project_slug}/api-keys`

The response is the **only** time you see the key. It returns the `ApiKey`
schema, which is `ApiKeyConcealed` plus `value`. Every listing
(`GET .../api-keys`) returns `ApiKeyConcealed` — no `value`. Capture it into
your secret store on the spot.

Give this key to CI. It is the one that can upload specifications; a personal
key cannot.

### 6. Rotate and revoke

- `POST .../api-keys/{api_key_id}` — update/rotate
- `DELETE .../api-keys/{api_key_id}` — revoke

Before revoking, check `last_used_at` on the listing to see whether anything is
still calling with it.

Personal keys live under `/v1/users/{user_id}/api-keys` with the same
create / update / delete shape.

### 7. Register a webhook

`POST /v1/organizations/{organization_slug}/projects/{project_slug}/webhooks`
with body `{ "endpoint": "https://example.com/api/webhooks" }`.

The response carries `secret` (prefixed `whsec_`) — again, once only; listings
return `WebhookConcealed`. You will receive:

```json
{ "id": "...", "object": "event", "type": "specification.created",
  "timestamp": 1740105435, "data": { /* Specification */ } }
```

`type` is `specification.created` or `specification.deleted`.

Toggle delivery with `is_enabled` via `POST .../webhooks/{webhook_id}`; remove
with `DELETE .../webhooks/{webhook_id}`.

> **Signature verification is not documented.** Hey API issues a signing secret
> but publishes no header name or hashing scheme, so you cannot currently
> verify a delivery from public documentation. Treat the endpoint as
> untrusted input, terminate TLS properly, and ask Hey API for the scheme.

## Rules that will bite you

- **Updates are `POST` to the item path**, not `PUT` or `PATCH` — except
  `/v1/users/{user_id}/waitlists/{waitlist_id}`, which really is `PUT`. The
  contract is inconsistent here; follow the spec per operation.
- **Secrets are returned exactly once.** `ApiKey.value` and `Webhook.secret`
  never appear again.
- **Skip `/v1/internal/*`.** Those paths are in the published document but are
  platform-internal service hooks (Clerk user sync, inbound webhook receivers),
  not consumer surface.
- **No idempotency and no `400`.** Repeat creates make duplicates; malformed
  input has no declared failure mode.

## Errors

`401` unauthenticated, `403` not permitted on this org/project, `404` unknown
record. Envelope: `{ "error": { "message", "request_id", "status",
"timestamp" } }`. See `errors/hey-api-problem-types.yml`.
