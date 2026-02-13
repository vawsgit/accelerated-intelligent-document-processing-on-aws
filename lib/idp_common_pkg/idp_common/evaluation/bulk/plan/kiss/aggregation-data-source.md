# KISS Decision #4: Aggregation Data Source

## Decision

Where does the `test_results_resolver` Lambda read per-document confusion matrices from, and what engine aggregates them?

| Option | Data Source | Aggregation Engine | Pros | Cons |
|--------|-----------|-------------------|------|------|
| A | S3 eval JSONs | Custom `BulkEvaluationAggregator` | Simple, no deps, works today | N S3 reads per test run |
| B | S3 eval JSONs | Stickler `aggregate_from_comparisons()` | Stickler owns the logic | Stickler refactor hasn't shipped; packaging problem (221 MB) |
| C | Athena SQL | SQL aggregation | Scales to 1000s, parallel | Can't replicate Stickler's confusion matrix exactly from existing schema |
| D | Consolidated parquet | Stickler or custom, via pandas | Single file read, portable | Requires writing consolidated file in reporting pipeline |

## Recommendation: Option A (S3 eval JSONs + custom aggregator) ✅

> **🔄 Updated 2026-02-12:** With [Stickler PR #74](https://github.com/awslabs/stickler/pull/74), the aggregation engine for `idp_common` is now Stickler's `aggregate_from_comparisons()`. The Lambda still uses the standalone shim. Data source (S3 eval JSONs) is unchanged.

This is the simplest path that works today:

1. Query Athena for document IDs in the test run (already done for existing metrics)
2. Read each eval JSON from S3 (N `GetObject` calls)
3. Feed confusion matrices into aggregator (Stickler-native in `idp_common`, standalone shim in Lambda)
4. Cache result in DynamoDB — subsequent requests served from cache

The DynamoDB cache means aggregation runs **once per test run**. The N S3 reads are a one-time cost.

## Why not the others?

**Option B (Stickler native):** ~~Blocked on Stickler shipping `aggregate_from_comparisons()`.~~ **Now available** via [PR #74](https://github.com/awslabs/stickler/pull/74) (merged to `dev`). Used in `idp_common` contexts. Still can't go into the bare Zip Lambda due to 221 MB dependency chain.

**Option C (Athena SQL):** The `attribute_evaluations` table has `matched` (bool) and `score` (float) per attribute, but NOT the raw confusion matrix breakdown. Stickler handles nested/list fields with Hungarian matching — can't replicate that in SQL. Would produce different numbers than Stickler.

**Option D (Consolidated parquet):** Elegant for notebooks and large-scale analysis, but adds complexity to the reporting pipeline and requires `pyarrow` in the Lambda. Good future enhancement — not needed for initial implementation since DynamoDB caching eliminates the repeated-read concern.

## Migration Path

1. **Now:** Option A — Stickler `aggregate_from_comparisons()` for `idp_common`, standalone shim for Lambda, S3 eval JSONs, DynamoDB cache
2. **Phase 1.5 (optional):** Add confusion matrix to reporting parquet for notebook use
3. ~~**After Stickler refactor:** Swap custom aggregator for Stickler's `aggregate_from_comparisons()`~~ → Done by design for `idp_common`; Lambda shim stays
4. **Future:** Consolidated parquet via `stickler-eval[storage]` for large-scale analysis
