# KISS: S3 Direct vs Athena Column for Confusion Matrix Retrieval

## Decision

How should the `test_results_resolver` Lambda retrieve per-document confusion matrices for bulk aggregation?

| | Option A: Athena Column | Option B: S3 Direct (Recommended ✅) |
|---|---|---|
| **Approach** | Add `confusion_matrix_json` string column to `document_evaluations` parquet schema, query via Athena | Query Athena for doc IDs only, read eval JSONs directly from S3 |
| **Schema change** | Yes — parquet schema + Glue table | None |
| **Backfill needed** | Yes — existing data lacks the column | No — works with existing eval JSONs (once Phase 1 adds confusion_matrix) |
| **Data paths** | Two: parquet pipeline + S3 eval JSONs | One: S3 eval JSONs |
| **SQL access** | ✅ Can query confusion matrices via Athena | ❌ Not available |
| **Speed** | Fast (single Athena query) | Slower (N S3 GetObject calls) |
| **Lambda timeout risk** | Low | Medium for >500 docs (mitigated by DynamoDB cache) |
| **Implementation time** | ~3 days | ~1.5 days |

## Recommendation

**Use S3 Direct (Option B) for initial implementation.**

The DynamoDB cache means aggregation only runs once per test run, so the S3 read overhead is a one-time cost. The Athena column can be added later if SQL-level access becomes a requirement.

## Steps (S3 Direct)

1. Phase 1: Ensure `confusion_matrix` is in eval results JSON saved to S3
2. Skip Phase 1c entirely (no parquet schema change)
3. Phase 3: Query Athena for `document_id` list, read each eval JSON from S3
4. Feed confusion matrices into aggregator (Stickler-native or Lambda shim)
5. Cache result in DynamoDB — subsequent requests served from cache

## Future: Option C — Confusion Matrix Parquet Files

A third option emerged from reviewer feedback: write confusion matrices to dedicated parquet files in the reporting bucket, then read a single consolidated file instead of N eval JSONs.

| Aspect | S3 Direct (current) | Parquet Files (future) |
|--------|---------------------|----------------------|
| Read pattern | N `GetObject` calls | 1 prefix scan + concat |
| Requires reporting pipeline change | No | Yes |
| Requires `pyarrow` in Lambda | No | Yes |
| Notebook experience | Load individual JSONs | `pd.read_parquet()` on prefix |

This is deferred because DynamoDB caching eliminates the repeated-read concern for the Lambda path. The parquet approach is most valuable for notebooks analyzing large test runs. See [kiss/aggregation-data-source.md](./aggregation-data-source.md) and [future-stickler-refactor.md](../future-stickler-refactor.md).
