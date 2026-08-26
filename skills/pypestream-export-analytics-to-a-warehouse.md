---
name: pypestream-export-analytics-to-a-warehouse
description: >-
  Configure, monitor, pause, backfill and tear down a Pypestream Analytics batch export to S3 or
  Snowflake, and read events, persons, cohorts and insights from a dataset.
api: pypestream:analytics-api
generated: '2026-08-26'
method: generated
source: >-
  openapi/pypestream-analytics-api-openapi.json,
  https://developers.pypestream.com/reference/overview-2
base_url: https://analytics.pypestream.com/
operations:
  - list
  - retrieve
  - events_list
  - persons_list
  - cohorts_list
  - cohorts_persons_retrieve
  - insights_trend_retrieve
  - insights_funnel_retrieve
  - insights_retention_retrieve
  - insights_path_retrieve
  - batch_exports_list
  - batch_exports_create
  - batch_exports_retrieve
  - batch_exports_update
  - batch_exports_partial_update
  - batch_exports_destroy
  - batch_exports_pause_create
  - batch_exports_unpause_create
  - batch_exports_backfill_create
  - batch_exports_runs_list
  - batch_exports_runs_retrieve
---

# Export Pypestream Analytics to a warehouse

Authentication is an **HTTP Bearer personal access token**, created at
`https://analytics.pypestream.com/me/settings`.

## Find your dataset

A **Dataset** is a distinct collection of analytics data, typically one environment (Production,
Testing, Development). `GET /api/datasets/` (`list`) enumerates the datasets you can reach;
`GET /api/datasets/{id}/` (`retrieve`) fetches one. Everything else is scoped under
`/api/datasets/{dataset_id}/`.

## Explore before you export

`events_list`, `persons_list`, `cohorts_list`, `actions_list`, `event_definitions_list`,
`property_definitions_retrieve`, `dashboards_list`, `insights_list` and `kpis_list` are for
**overview and exploration**. The Events endpoint says so explicitly — for moving data, use batch
exports, not this.

Filtering on `events_list`: `after`, `before`, `event`, `distinct_id`, `person_id`, `properties`,
and the **experimental** `select` / `where` (JSON-serialized PypeQL arrays). `maskPii=true` masks
personally identifiable information in the response — it is **off by default**, so set it
deliberately when a human or a model will see the output.

Pagination is `limit` / `offset` with a `{count, next, previous, results}` envelope; `next` and
`previous` are absolute URIs.

## Set up the export

`POST /api/datasets/{dataset_id}/batch_exports/` (`batch_exports_create`) needs a destination type
(S3 or Snowflake), an interval (e.g. daily, hourly) and credentials. **Snowflake support is in
beta.** Pypestream publishes a recommended Snowflake DDL:

```sql
CREATE TABLE your_table_name (
    uuid VARCHAR(36),
    timestamp TIMESTAMP_NTZ,
    created_at TIMESTAMP_NTZ,
    event VARCHAR,
    properties VARIANT,
    distinct_id VARCHAR(36),
    person_id VARCHAR(36),
    person_properties VARIANT,
    elements_chain VARCHAR
);
```

## Operate it

- **Monitor**: `batch_exports_runs_list` for execution history, `batch_exports_runs_retrieve` for a
  single run — use these to debug failures and verify volume.
- **Pause / resume**: `batch_exports_pause_create` stops future runs while the configuration
  **remains saved**; `batch_exports_unpause_create` restarts on the configured interval. This pair
  is fully symmetric and is the safe way to stop an export.
- **Repair a gap**: `batch_exports_backfill_create` triggers a backfill for a specific historical
  date range — data from before the export existed, or from a period while it was paused. This is
  the compensating action after a pause.
- **Change**: `batch_exports_update` (PUT) **replaces the whole configuration**;
  `batch_exports_partial_update` (PATCH) changes named fields. Prefer PATCH unless you mean to
  replace.

## Rules an agent must not get wrong

- **`batch_exports_destroy` is permanent.** It stops all future runs and removes the setup. If you
  only want to stop the flow, **pause instead** — pause is reversible, delete is not, and Pypestream
  publishes no undelete and no grace period.
- **No window is published for backfill.** Pypestream states no maximum lookback, so do not promise
  a user that a gap of a given age is recoverable — check by running a backfill for a narrow range
  first.
- **This contract declares no errors at all.** None of its 43 operations declares a 4xx or 5xx
  response. Do not assume a failure shape; handle any non-2xx defensively and log the raw body.
- **No idempotency key.** A retried `batch_exports_create` after a timeout may create a second
  export. List before you create.
