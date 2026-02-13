# Future: Stickler Refactor & Integration

> **🔄 Updated 2026-02-12:** [Stickler PR #74](https://github.com/awslabs/stickler/pull/74) merged `aggregate_from_comparisons()`, `update_from_comparison_result()`, and optional `target_schema` into Stickler's `dev` branch. Steps 1 and 2 below are now actionable once a Stickler release ships. See [.working/pr-74-integration/research.md](../.working/pr-74-integration/research.md) for full analysis.

## Current State

~~The IDP Accelerator uses a custom `BulkEvaluationAggregator` (~80 lines, pure Python) that replicates Stickler's `_accumulate_confusion_matrix()` logic.~~

**Updated:** With PR #74 merged, `idp_common` will use `from stickler import aggregate_from_comparisons` directly. The only custom aggregation code is the standalone `bulk_aggregator.py` shim in the `test_results_resolver` Lambda (bare Zip, can't import Stickler's 221 MB deps).

## Stickler Change #1: `aggregate_from_comparisons()` — ✅ SHIPPED (PR #74)

Shipped as a standalone module-level function:

```python
from stickler import aggregate_from_comparisons

# Takes list of compare_with() result dicts, returns ProcessEvaluation
result = aggregate_from_comparisons(comparison_results)
```

Also shipped: `update_from_comparison_result()` as an instance method on `BulkStructuredModelEvaluator` for incremental accumulation, and `target_schema` is now optional.

## Stickler Change #2: `stickler-eval[storage]` optional dependency

**Priority: Low** — convenience enhancement, not blocking. **Status: Not yet shipped.**

Add optional S3/MinIO support for saving/loading evaluation state as parquet:

```toml
# pyproject.toml
[project.optional-dependencies]
llm = ["strands-agents>=1.0.0,<=1.16.0"]
storage = ["boto3>=1.26.0", "pyarrow>=14.0.0"]    # ← NOT YET SHIPPED
```

## Migration Path for IDP Accelerator

### Step 1 (current): Standalone Lambda shim + Stickler for idp_common
- `test_results_resolver/bulk_aggregator.py` — standalone zero-dep shim for Lambda
- `idp_common/evaluation/bulk/__init__.py` — re-exports `aggregate_from_comparisons` from Stickler
- Notebooks and CLI use Stickler directly

### ~~Step 2 (after Stickler ships `aggregate_from_comparisons`):~~ → NOW ACTIONABLE
- ~~Replace `BulkEvaluationAggregator` in `idp_common`~~ → Done by design (no custom class in `idp_common`)
- Bump `stickler-eval` version in `pyproject.toml` to the release containing PR #74
- Lambda shim stays as custom code (packaging constraint unchanged)

**Blocking:** Awaiting a tagged Stickler release that includes PR #74. Currently on `dev` only.

### Step 3 (after `stickler-eval[storage]` ships):
- Notebooks use Stickler's native parquet save/load
- Reporting pipeline optionally writes consolidated parquet
- Lambda path unchanged (still reads S3 eval JSONs, cached in DynamoDB)

### Step 4 (long-term): Eliminate Lambda shim
- If Stickler extracts accumulation logic into a zero-dep subpackage (e.g., `stickler-eval[core]` without scipy/pandas), the Lambda shim can be replaced with a direct import
- Or: if the Lambda is converted to Docker image for other reasons, Stickler can be imported directly
