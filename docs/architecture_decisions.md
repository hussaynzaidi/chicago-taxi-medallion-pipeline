# Architecture Decision Record — Chicago Taxi Medallion Pipeline

## Decision 1: Bronze is append-only, never overwritten

**Problem:** Raw source data contains errors. If we transform during ingest and later
discover the transforms were wrong, we have no way to replay from the original.

**Decision:** Bronze receives raw data exactly as received. The only addition is an
`ingested_at` audit column. No type casting, no filtering, no joins. This layer is
immutable by convention.

**Consequence:** Silver and Gold can always be fully rebuilt from Bronze. Recovery
from any downstream bug is a re-run of notebooks 02 and 03 — Bronze is never touched.

---

## Decision 2: Silver uses MERGE INTO, not overwrite

**Problem:** Running a pipeline twice with overwrite semantics drops and recreates
the table, which is unsafe if Bronze was only partially loaded or if upstream systems
retry failed writes.

**Decision:** Silver uses Delta Lake's MERGE INTO pattern, matching on `trip_id`.
Existing rows are updated if the source data changed; new rows are inserted.

**Consequence:** The Silver notebook is fully idempotent — running it 10 times
produces the same result as running it once. This is a required property for any
pipeline running on a schedule.

---

## Decision 3: Data quality is logged, not just applied

**Problem:** Silently dropping bad rows means data consumers never know how clean
their data is. A 30% rejection rate is a business problem, not just a pipeline detail.

**Decision:** Before cleaning, the pipeline measures null counts, negative values,
and duplicate trip IDs. These metrics are written to a `_dq_log` Delta table with
a run timestamp. Stakeholders can query DQ history to track data quality trends.

**Consequence:** The DQ log is queryable. A downstream analyst who notices
anomalies can check `SELECT * FROM chicago_taxi.dq_log ORDER BY run_timestamp DESC`
and immediately see whether this is a pipeline problem or a source data problem.

---

## Decision 4: Gold uses window functions, not just GROUP BY

**Problem:** Simple aggregations answer "what happened" but not "is this trend
improving or worsening." Analysts need rolling context to interpret daily numbers.

**Decision:** The `gold_rolling_fare_by_area` table uses a 7-day window function
(`ROWS BETWEEN -6 AND CURRENT ROW`) partitioned by community area. This gives
analysts a trend signal without requiring them to write window logic in their
BI tool — which is typically slower and harder to maintain.

---

## Decision 5: ZORDER on the highest-cardinality filter column

**Problem:** BI tools filter Gold tables by `pickup_community_area` in most
queries. Without data layout optimisation, Spark scans all files on each query.

**Decision:** After each Gold write, we run `OPTIMIZE ... ZORDER BY (pickup_community_area)`.
This co-locates rows with the same community area value on the same set of files,
drastically reducing the number of files scanned per query.

---

## What would change in a paid production environment?

| This portfolio build | Production equivalent |
|---|---|
| Databricks Community Edition | Databricks Standard or Premium on Azure |
| DBFS storage | Azure Data Lake Storage Gen2 |
| Manual monthly download | Azure Data Factory or Event Hubs ingest |
| Email failure alert | PagerDuty or Azure Monitor integration |
| Single cluster | Job clusters (auto-terminate after each run) |

The code is identical. The mount path changes from `/mnt/chicago-taxi/` to
`abfss://container@storageaccount.dfs.core.windows.net/chicago-taxi/`.