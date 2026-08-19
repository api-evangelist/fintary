---
name: Onboard a Fintary agent and place them in the hierarchy
description: >-
  Create an agent in Fintary, attach them to an upline so commissions roll up correctly, and
  verify the placement. Use when a new producer joins a brokerage or carrier distribution
  channel and must be payable before the next commission run.
api: openapi/fintary-open-api-openapi.yml
operations:
  - agents.create
  - agents.assignUpline
  - agents.get
  - agents.removeUpline
generated: '2026-08-14'
method: generated
source: >-
  Grounded in openapi/fintary-open-api-openapi.yml, harvested from
  https://api.fintary.com/openapi-doc on 2026-08-14. Every operationId below appears
  verbatim in that document.
---

# Onboard a Fintary agent and place them in the hierarchy

Hierarchy placement is the whole game in insurance commission management: an agent with no
upline is payable but their overrides do not roll up, so the run reconciles wrong. Create the
agent and place them in the same unit of work.

## Before you start

- Base URL is `https://api.fintary.com`. Every path below is prefixed `/openapi/`.
- Send `x-api-key: <key>` or `Authorization: Bearer <token>`. Keys are not self-service —
  they come from a Fintary representative against your tenant. An unauthenticated call
  returns `401 {"success":false,"data":null,"message":"Missing or invalid API key","statusCode":401}`.
- **There is no idempotency key.** A retried create makes a second agent. Read step 1's
  guard before you retry anything.
- There is no sandbox. Whatever you call, you call in production.

## Steps

1. **Check the agent does not already exist.** Call `agents.list`
   (`GET /openapi/agents`) filtered with `company_name`, `status` or `type`, and page with
   `page` (0-based) and `limit` (1–1000). Do this even on a retry — because the API has no
   idempotency contract, this list call is the only thing standing between a network timeout
   and a duplicate producer record.

2. **Create the agent.** Call `agents.create` (`POST /openapi/agents`). Keep the identifier
   you will later sync from your own system of record: Fintary's SSO resolution reads an
   agent `sync_id`, so populating it now is what lets that producer sign in later without a
   manual match.

3. **Capture the returned `strId`.** Records across both Fintary APIs are keyed by a string
   `strId`/`str_id`. Store it against your own record; you will need it for every subsequent
   call and for the AMS API.

4. **Assign the upline.** Call `agents.assignUpline`
   (`POST /openapi/agents/{id}/assign-upline`) with the new agent's `strId`. Until this
   succeeds, nothing overrides up the chain.

5. **Verify.** Call `agents.get` (`GET /openapi/agents/{id}`) and confirm the record and its
   placement. Do not treat the create response as confirmation of the upline — they are two
   calls and the second can fail on its own.

6. **Correcting a bad placement.** Call `agents.removeUpline`
   (`DELETE /openapi/agents/{id}/assign-upline`). Note the spec's own wording: removal is
   addressed **by `contact_hierarchy` str_id**, not by the agent id you used to assign, so
   read the hierarchy record before deleting.

## Error handling

Four envelopes are in circulation across Fintary's surface. Parse defensively:

| Shape | Where |
|---|---|
| `{error, code}` | Open API coded errors, e.g. `FORBIDDEN_WIDGET_CONTEXT` |
| `{error}` | Open API simple errors, e.g. `Account not found` |
| `{code, message}` | AMS registry routes, closed enum |
| `{success, data, message, statusCode}` | observed live on 401; **no published schema describes it** |

- `400` — validation. Fix the payload; do not retry unchanged.
- `401` — missing or invalid credential.
- `403` — your key is scoped to a different account. The `account_id` override parameter is
  documented as Fintary Admin or Account Admin only; do not reach for it to work around this.
- `404` — agent not found. On `assign-upline`, this usually means the *upline* id is wrong.
- `500` — retry once, then stop and reconcile with `agents.list` before trying again, since
  there is no idempotency key to protect you.

## What this API will not do for you

- No `429` is declared on any operation and no `RateLimit-*` or `Retry-After` header is
  returned. Impose your own back-off; there is no runtime signal.
- No webhook or event fires on agent creation. If another system needs to know, you tell it.
