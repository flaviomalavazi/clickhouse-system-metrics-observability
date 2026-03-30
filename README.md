# ClickHouse Cloud Monitoring Dashboard

This repo sets up a self-contained observability layer for ClickHouse Cloud services. It consolidates system telemetry from all cluster replicas into a single queryable database and ships a ready-to-import dashboard with 12 panels covering query performance, insert health, and infrastructure metrics.

## How it works

ClickHouse Cloud runs multiple replicas, each writing to its own local system tables (`system.query_log`, `system.metric_log`, `system.asynchronous_metric_log`). These tables are not directly unified — different replicas may run different ClickHouse versions with slightly different schemas.

This repo solves that by:

1. **Creating a consolidated database** (`clickstack_consolidated`) with normalized table definitions and a 30-day TTL.
2. **Doing a one-time historical backfill** from all replicas via `merge()` table function calls, handling schema divergence across replica generations.
3. **Setting up incremental materialized views** so new data flows in automatically going forward.

## Files

| File | Purpose |
|------|---------|
| `create_tables_consolidated_metrics.sql` | Creates the `clickstack_consolidated` database and its three tables, then backfills them from the system tables of all replicas. Run once. |
| `create_incremental_materialized_views.sql` | Creates materialized views that continuously forward new rows from `system.*` into `clickstack_consolidated.*`. Run once after the backfill. |
| `periodic_inserts.sql` | Alternative to materialized views: incremental INSERTs that append only rows newer than the current max `event_time` in each consolidated table. Use this if you prefer scheduled jobs over MVs. |
| `clickhouse-dashboard.json` | Dashboard definition importable into the ClickHouse built-in dashboard UI. |

## Consolidated tables

### `clickstack_consolidated.query_log`
Mirrors `system.query_log` across all replicas. Used for query latency percentiles, per-table query counts, slow query analysis, and error rates.

### `clickstack_consolidated.metric_log`
Mirrors `system.metric_log` across all replicas. Used for CPU, memory, disk I/O, S3 request counts, and network throughput.

### `clickstack_consolidated.asynchronous_metric_log`
Mirrors `system.asynchronous_metric_log` across all replicas. Provides the time-series backbone for infrastructure metrics.

All three tables partition by `toYYYYMM(event_date)` and have a 30-day TTL with `ttl_only_drop_parts = 1`.

## Schema divergence handling

ClickHouse Cloud manages rolling upgrades, so older replicas expose versioned system tables (`query_log_1`, `query_log_2`, …) with schemas from an earlier ClickHouse version. The backfill script handles two distinct cases:

- **`metric_log`**: The source tables across replicas collectively expose 1572 columns, while the consolidated table defines 1556. The INSERT uses an explicit column list to select only the columns present in the target schema.
- **`query_log`**: Two columns (`authenticated_user`, `is_internal`) were added in a newer ClickHouse release and are absent from older replica tables. The backfill uses a `UNION ALL` to preserve real values from current-version replicas (matched via `^query_log$`) while falling back to empty defaults for older ones (matched via `query_log_.*`).

## Dashboard

`clickhouse-dashboard.json` can be imported directly into the ClickHouse Cloud dashboard UI. It targets the `clickstack_consolidated` database and contains 12 panels across three sections:

**Selects**
- Query Latency — p50 / p90 / p99 over time
- Query Count by Table — SELECT volume per table
- Most Time Consuming Query Patterns — aggregated by normalized query hash
- Slowest Queries — individual query drill-down

**Inserts**
- Insert Queries Per Table — INSERT throughput over time
- Max Active Parts per Partition — merge pressure indicator
- Active Parts Per Partition — per-partition part count table

**Infrastructure**
- CPU Usage (cores)
- Memory Usage
- Disk I/O
- S3 Requests
- Network (receive/send bytes)

## Usage

### Prerequisites

- `clickhouse-client` installed locally
- A `.env` file in this directory with `CLICKHOUSE_HOST`, `CLICKHOUSE_USER`, `CLICKHOUSE_PASSWORD`, and `CLICKHOUSE_DATABASE`

### One-time setup

```bash
# 1. Create tables and backfill historical data from all replicas
clickhouse client --host "$CLICKHOUSE_HOST" --user "$CLICKHOUSE_USER" \
  --password "$CLICKHOUSE_PASSWORD" --secure \
  --multiquery < create_tables_consolidated_metrics.sql

# 2. Set up materialized views for continuous ingestion
clickhouse client --host "$CLICKHOUSE_HOST" --user "$CLICKHOUSE_USER" \
  --password "$CLICKHOUSE_PASSWORD" --secure \
  --multiquery < create_incremental_materialized_views.sql
```

### Incremental refresh (alternative to MVs)

If you prefer scheduled jobs over materialized views, run `periodic_inserts.sql` on a cron schedule. Each statement appends only rows newer than the current max `event_time` in the consolidated table, making it safe to run repeatedly.

### Import the dashboard

In ClickHouse Cloud, navigate to **Dashboards → Import** and upload `clickhouse-dashboard.json`. Set the data source connection to your ClickHouse Cloud service.
