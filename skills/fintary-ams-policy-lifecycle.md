---
name: Manage a policy through the Fintary AMS API
description: >-
  Create a policy, attach customers and a servicing team, move it through status, run its
  task list, and file documents into the policy document repository. Use when Fintary is the
  agency management system of record for the book.
api: openapi/fintary-ams-api-openapi.yml
operations:
  - POST /api/ams/policies
  - GET /api/ams/policies/{strId}
  - PATCH /api/ams/policies/{strId}
  - PATCH /api/ams/policies/{strId}/status
  - GET /api/ams/policies/{strId}/customers
  - GET /api/ams/policies/{strId}/team
  - PATCH /api/ams/policies/{strId}/team
  - PATCH /api/ams/policies/{strId}/case-manager
  - GET /api/ams/policies/{strId}/commission
  - GET /api/ams/policies/{strId}/tasks
  - POST /api/ams/policies/{strId}/tasks
  - POST /api/ams/policies/{strId}/documents
  - POST /api/ams/policies/{strId}/documents/finalize
  - GET /api/ams/policies/{strId}/documents
  - GET /api/ams/policies/{strId}/documents/{documentStrId}/download
  - POST /api/ams/policies/list
  - GET /api/ams/policies/summary
generated: '2026-08-14'
method: generated
source: >-
  Grounded in openapi/fintary-ams-api-openapi.yml, harvested from
  https://api.fintary.com/api-doc on 2026-08-14. That document declares NO operationIds on
  any of its 60 operations, so every operation above is identified by method and path exactly
  as the spec publishes it. Nothing is invented.
---

# Manage a policy through the Fintary AMS API

The AMS API is the record core — policies, customers, agents, contracts, tasks and documents.
It is a different contract from the Open API: bearer token only, `/api/ams/` prefix, no
operationIds, and a thinner declared error surface.

## Before you start

- Base URL `https://api.fintary.com`, paths prefixed `/api/ams/`.
- **Bearer token only.** `Authorization: Bearer <token>`. The `x-api-key` scheme is declared
  on the Open API document and not on this one. An unauthenticated call returns `401` with
  the plain-text body `Not authenticated` — no JSON, so do not assume a parseable envelope.
- **The spec understates errors.** Only `200`, `201`, `304` and `404` are declared across all
  60 operations. No `400`, `401` or `403` appears anywhere, yet `401` is returned live and
  `AmsRegistryErrorSchema` enumerates `unauthenticated`, `permission_denied`,
  `invalid_argument` and `failed_precondition`. Handle those four regardless of what the
  document declares.
- No idempotency key exists. Every create below can duplicate on retry.

## Steps

1. **Create the policy.** `POST /api/ams/policies`. Capture the returned `strId` — it keys
   every subsequent call.

2. **Attach the customer side.** `GET /api/ams/policies/{strId}/customers` to read the
   associated customers. Create customers first if they do not exist
   (`POST /api/ams/customers`), then `GET /api/ams/customers/{strId}/policies` from the other
   direction to confirm the link took.

3. **Set the servicing team.** `PATCH /api/ams/policies/{strId}/team` and, separately,
   `PATCH /api/ams/policies/{strId}/case-manager`. These are two distinct calls; setting the
   team does not set the case manager. Read back with `GET /api/ams/policies/{strId}/team`.

4. **Move status deliberately.** `PATCH /api/ams/policies/{strId}/status` is its own endpoint,
   not a field on the general update. Use it for status; use `PATCH /api/ams/policies/{strId}`
   for everything else. Valid statuses come from `GET /api/ams/configs/statuses` — read that
   rather than hardcoding strings.

5. **Work the task list.** `GET /api/ams/policies/{strId}/tasks` and
   `POST /api/ams/policies/{strId}/tasks`. Task-level updates live on the sibling routes
   `PATCH /api/ams/tasks/{taskId}` and `POST /api/ams/tasks/{taskId}/escalate`. Use
   `GET /api/ams/tasks/scope` to learn which agents the current user may act for (self plus
   downline contacts) before you assign anything.

6. **File documents — this is a two-phase upload.**
   - Phase 1: `POST /api/ams/policies/{strId}/documents` starts the upload.
   - Phase 2: `POST /api/ams/policies/{strId}/documents/finalize` completes it.
   A document that is started and never finalized is not filed. If phase 2 fails, do **not**
   re-run phase 1 blind — list with `GET /api/ams/policies/{strId}/documents` first, because
   there is no idempotency key and a second phase 1 will orphan an upload.
   - Retrieve with `GET /api/ams/policies/{strId}/documents/{documentStrId}/download`, which
     returns a **short-lived signed URL**. Fetch it immediately; do not persist or share it.

7. **Read the money view.** `GET /api/ams/policies/{strId}/commission` returns receivables,
   payouts by role and statement totals for the policy — the AMS-side counterpart to the Open
   API's agent-centric commission reads.

## Listing and filtering

- `POST /api/ams/policies/list` is a **POST** that lists. Pagination lives inside the request
  body on this route, not in query parameters — the AMS API exposes only `entity` and `query`
  as query parameters.
- `GET /api/ams/policies/filters` returns filter metadata and
  `POST /api/ams/policies/filter-options` returns the option values. Build filter UI from
  those two rather than guessing field names.
- `GET /api/ams/policies/summary` returns counts grouped by status.
- `PATCH /api/ams/policies/bulk-update` exists for fan-out edits. With no idempotency key, a
  retried bulk update is the highest-blast-radius mistake available on this API. Confirm the
  first attempt's outcome before sending a second.

## The capabilities registry

`GET /api/ams/registry/capabilities` returns Fintary's own AI/wire manifest for an AMS entity:
bindable fields, widget vocabulary, endpoint allowlist and limits. If you are driving this API
from an agent, read it first — it is Fintary telling you what it considers bindable. It is
authenticated and per-tenant, so it is not a public discovery document. Companion routes:
`GET /api/ams/registry/page-config`, `GET /api/ams/registry/page-configs`,
`PUT /api/ams/registry/page-config-save`, `PUT /api/ams/registry/page-config-default`.

## Error handling

| Envelope | Where |
|---|---|
| `{code, message}` with `code` in `invalid_argument`, `not_found`, `failed_precondition`, `unauthenticated`, `permission_denied` | AMS registry routes |
| plain text `Not authenticated` | observed live on `401` |

- `304` is declared on two operations — honour conditional responses rather than treating a
  non-200 as failure.
- `404` on a `{strId}` route means the record is gone or was never yours; do not retry.
- No `429` is declared and no rate-limit headers are returned. Self-throttle, especially on
  bulk routes.

## What this API will not do for you

- No webhooks and no AsyncAPI. Status changes, task completion and document finalization emit
  nothing — poll or drive the change yourself.
- No sandbox. There is no test tenant, no test mode and no fixture data.
