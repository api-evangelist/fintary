---
name: Assemble an agent's compensation picture
description: >-
  Pull one producer's commissions, payouts, policies and dashboard snapshot from Fintary and
  reconcile them into a single statement. Use when answering "why was I paid this?",
  preparing a producer review, or investigating a commission dispute.
api: openapi/fintary-open-api-openapi.yml
operations:
  - agents.get
  - agents.listCommissions
  - agents.listPayouts
  - agents.listPolicies
  - agents.getDashboard
  - compReports.getData
generated: '2026-08-14'
method: generated
source: >-
  Grounded in openapi/fintary-open-api-openapi.yml, harvested from
  https://api.fintary.com/openapi-doc on 2026-08-14. Every operationId below appears
  verbatim in that document.
---

# Assemble an agent's compensation picture

Fintary splits a producer's compensation across four read endpoints plus a report surface.
None of them is a statement on its own — the statement is the join, and the join is yours to
make.

## Before you start

- Base URL `https://api.fintary.com`, paths prefixed `/openapi/`.
- Send `x-api-key: <key>` or `Authorization: Bearer <token>`.
- Every call here is a read. There is no idempotency contract, but there is nothing to
  duplicate either — retries are safe.
- All five reads are paginated and date-filtered independently. Use the **same** window on
  each or the numbers will not reconcile.

## Steps

1. **Resolve the agent.** `agents.get` (`GET /openapi/agents/{id}`). Confirm you have the
   right producer before you pull money data. If you only have a name, get there via
   `agents.list` (`GET /openapi/agents`) with the `company_name`, `status` or `type` filters.

2. **Fix your window once.** Choose `start_date` and `end_date` (ISO 8601) and reuse them
   verbatim across steps 3–5. Mismatched windows are the single most common cause of a
   statement that does not tie out.

3. **Pull earned commissions.** `agents.listCommissions`
   (`GET /openapi/agents/{id}/commissions`). This is what the agent *earned* in the window.

4. **Pull payouts.** `agents.listPayouts` (`GET /openapi/agents/{id}/payouts`). This is what
   the agent was *paid*. Commissions and payouts do not have to match: advances,
   chargebacks and payment timing all move money between the two, and explaining that gap is
   usually the actual question being asked.

5. **Pull the book.** `agents.listPolicies` (`GET /openapi/agents/{id}/policies`) to attribute
   each line to a policy.

6. **Take the snapshot for headline figures.** `agents.getDashboard`
   (`GET /openapi/agents/{id}/dashboard`). Treat this as the summary the producer sees in
   their portal — use it to sanity-check your join, not as the source of the line items.

7. **Reconcile at the account level if the numbers disagree.** `compReports.getData`
   (`GET /openapi/comp-reports/data`) returns commission report data across the account.

## Pagination

`page` is **0-based**. `limit` is 1–1000 on the agents list. Read every page — a partial read
silently understates a statement, which is worse than an error. Response envelopes expose
`pageRowCount` and `rowCount`; compare accumulated rows against `rowCount` before you call the
pull complete.

## Scoping and impersonation

`contact_id` and `account_id` query parameters appear on several of these routes and are
documented as **Fintary Admin or Account Admin** overrides — `contact_id` scopes rows to a
single contact, `account_id` overrides the account. Some routes require the two to be used
together. If you are building a producer-facing view, do not pass them: let the credential
scope the response. Passing an override you are not entitled to returns `403` with
`{error, code}`.

## Error handling

- `401` — missing or invalid credential.
- `403` — override parameter not permitted for your key, or wrong account.
- `404` — agent not found; verify with step 1 rather than retrying.
- `500` — retry with back-off. No `Retry-After` is returned, so choose your own.

## Limits to design around

- No `429` is declared and no rate-limit headers are returned. If you are fanning out across
  a whole book of producers, throttle yourself.
- There is no event surface — no webhooks, no AsyncAPI. A statement that must stay current
  has to be re-pulled on your schedule.
