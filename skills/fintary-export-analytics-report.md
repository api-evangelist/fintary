---
name: Query and export a Fintary analytics report or dataset
description: >-
  Discover published analytics reports and datasets, query them with filters and grouping,
  and export large results as CSV through the background-task queue. Use when feeding a
  warehouse, building a recurring commission report, or pulling a dataset too large for a
  synchronous response.
api: openapi/fintary-open-api-openapi.yml
operations:
  - analytics.listReports
  - analytics.getReportData
  - analytics.listDatasets
  - analytics.getDatasetData
  - analytics.getExportTaskStatus
  - analytics.listWidgets
  - analytics.listDefaultWidgets
  - analytics.restoreDefaultWidget
generated: '2026-08-14'
method: generated
source: >-
  Grounded in openapi/fintary-open-api-openapi.yml, harvested from
  https://api.fintary.com/openapi-doc on 2026-08-14. Every operationId below appears
  verbatim in that document.
---

# Query and export a Fintary analytics report or dataset

Fintary exposes two analytics surfaces that look similar and are not: **reports** are
published, saved configurations; **datasets** are the queryable tables underneath. Pick the
one that matches the question, then decide synchronous or background before you send
anything.

## Before you start

- Base URL `https://api.fintary.com`, paths prefixed `/openapi/`.
- Send `x-api-key: <key>` or `Authorization: Bearer <token>`.
- Reads are safe to retry. There is no idempotency contract and none is needed here.

## Choosing a surface

| You want | Use |
|---|---|
| A report someone already published | `analytics.listReports` then `analytics.getReportData` |
| Ad-hoc query over a table | `analytics.listDatasets` then `analytics.getDatasetData` |
| What the dashboard shows | `analytics.listWidgets` / `analytics.listDefaultWidgets` |

## Steps — published reports

1. `analytics.listReports` (`GET /openapi/analytics/reports`) to find the report.
2. `analytics.getReportData` (`GET /openapi/analytics/reports/{id}`).
   - **Pass the string `str_id`.** Integer IDs are still accepted for legacy compatibility
     but the spec marks them deprecated; do not build on them.
   - Window with `start_date` / `end_date` (`YYYY-MM-DD`), page with `page` (0-based) and
     `page_size`.
   - `metadata_only=true` returns the report configuration with no rows — call this first if
     you need to know the column shape before committing to a full pull.
   - `contact_id` scopes rows to a single contact and is a Fintary Admin / Account Admin
     override. Omit it unless you are entitled to it.

## Steps — datasets

1. `analytics.listDatasets` (`GET /openapi/analytics/datasets`).
2. `analytics.getDatasetData` (`GET /openapi/analytics/datasets/{name}`), which is the richer
   query surface:
   - `columns` — repeat the parameter once per column you want back.
   - `filters` — a JSON-encoded array of `{ column, operation, values }`.
   - `measures` — a JSON-encoded array of `{ column, aggregation, outputName }`, paired with
     `group_bys` when you aggregate.
   - `order_by` (repeatable for multi-column sort) with a matching repeated `order`, or
     `sort_model` as a JSON-encoded AG Grid sort model.
   - `filter_model` — a JSON-encoded AG Grid filter model, if you prefer the grid shape.
     Note the document models this explicitly: `AgGridFilterModelSchema`,
     `AgGridTextFilterSchema`, `AgGridDateFilterSchema` and their composite AND/OR variants
     are first-class schemas, so the public contract is coupled to that grid library.
   - `{column}_start` / `{column}_end` — dynamic per-column date-range bounds; substitute the
     real column name for `{column}`.
   - `limit` is 1–10000 with a **default of 50**. The default is small enough that a naive
     call looks like a complete answer and is not. Always set it.

## Exporting

- `csv_output=true` streams the result as a CSV attachment on the same request.
- For anything large, add `background_task=true` (it requires `csv_output=true`). The export
  is queued rather than returned.
- Poll `analytics.getExportTaskStatus` (`GET /openapi/analytics/tasks/{taskId}`) until it
  completes. There is no webhook and no callback — polling is the only mechanism.
- `email_account_admins=true` emails account admins when a background export finishes. This
  sends real mail to real people; do not set it on a test run.

Choose the background path deliberately. Fintary publishes no request-timeout figure and no
synchronous row ceiling, so the safe rule is: anything you would not want to hold a
connection open for goes through the queue.

## Widgets

`analytics.listWidgets` (`GET /openapi/analytics/widgets`) lists dashboard widgets;
`adminOnly=true` restricts to account-admin widgets and `contact_id` resolves widgets for a
specific contact under admin impersonation. `analytics.listDefaultWidgets`
(`GET /openapi/analytics/widgets/defaults`) lists the defaults, and
`analytics.restoreDefaultWidget` (`POST /openapi/analytics/widgets/defaults/restore`) puts one
back. The restore is the only write in this skill — and it has **no idempotency key**, so
send it once and check the result rather than retrying blind.

## Error handling

- `400` — malformed `filters`, `measures`, `filter_model` or `sort_model` JSON. These are
  strings inside a query parameter, so a bad escape surfaces here rather than as a 500.
- `401` — missing or invalid credential.
- `403` — `{error, code}`, e.g. `FORBIDDEN_WIDGET_CONTEXT` when a widget context is not yours.
- `404` — report or dataset name not found; confirm with the list operation.
- `500` — retry with self-imposed back-off; no `Retry-After` is returned.
