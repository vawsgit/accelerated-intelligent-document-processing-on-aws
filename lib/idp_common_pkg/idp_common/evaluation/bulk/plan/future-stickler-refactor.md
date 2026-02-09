# Future: Stickler Refactor & Integration

This document captures the planned Stickler changes and the migration path from the custom `BulkEvaluationAggregator` to Stickler-native aggregation.

## Current State

The IDP Accelerator uses a custom `BulkEvaluationAggregator` (~80 lines, pure Python) that replicates Stickler's `_accumulate_confusion_matrix()` logic. This exists because:

1. `BulkStructuredModelEvaluator.update()` takes `StructuredModel` instances — API mismatch (we have pre-computed confusion matrix dicts)
2. Stickler + deps = 221 MB — can't add to the bare Zip Lambda
3. The accumulation math is ~80 lines of textbook P/R/F1 derivation

## Stickler Change #1: `aggregate_from_comparisons()`

**Priority: High** — unblocks native Stickler aggregation in IDP

Add a method that accepts pre-computed `compare_with()` results:

```python
class BulkStructuredModelEvaluator:
    # Existing
    def update(self, gt_model: StructuredModel, pred_model: StructuredModel, doc_id=None): ...

    # NEW
    def aggregate_from_comparisons(self, comparisons: list[dict]) -> ProcessEvaluation:
        """Aggregate from pre-computed compare_with() results."""
        for comparison in comparisons:
            if "confusion_matrix" in comparison:
                self._accumulate_confusion_matrix(comparison["confusion_matrix"])
                self._processed_count += 1
        return self.compute()
```

This is a small refactor — it exposes the existing `_accumulate_confusion_matrix()` internal through a public API that accepts dicts instead of StructuredModel instances.

**No new dependencies required.**

## Stickler Change #2: `stickler-eval[storage]` optional dependency

**Priority: Low** — convenience enhancement, not blocking

Add optional S3/MinIO support for saving/loading evaluation state as parquet:

```toml
# pyproject.toml
[project.optional-dependencies]
llm = ["strands-agents>=1.0.0,<=1.16.0"]
storage = ["boto3>=1.26.0", "pyarrow>=14.0.0"]    # ← NEW
```

```python
# Usage
evaluator = BulkStructuredModelEvaluator(target_schema=InvoiceModel)
evaluator.save_results("s3://bucket/test-run-123/bulk_eval.parquet")
evaluator.save_results("file:///tmp/bulk_eval.parquet")
# MinIO
evaluator.save_results("s3://bucket/eval.parquet", endpoint_url="http://localhost:9000")
```

Follows the existing pattern from `comparators/llm.py`:
```python
try:
    import boto3
    BOTO3_AVAILABLE = True
except ImportError:
    BOTO3_AVAILABLE = False
```

## Migration Path for IDP Accelerator

### Step 1 (current PR): Custom aggregator
- `BulkEvaluationAggregator` in `idp_common/evaluation/bulk/aggregator.py`
- Copy as `test_results_resolver/bulk_aggregator.py` for Lambda use
- No Stickler dependency in the aggregation path

### Step 2 (after Stickler ships `aggregate_from_comparisons`):
- Replace `BulkEvaluationAggregator` in `idp_common` with:
  ```python
  from stickler.structured_object_evaluator.bulk_structured_model_evaluator import BulkStructuredModelEvaluator
  evaluator = BulkStructuredModelEvaluator(target_schema=None)
  evaluator.aggregate_from_comparisons(confusion_matrices)
  ```
- Lambda copy (`bulk_aggregator.py`) stays as custom code (no Stickler in Lambda)
- Or: if Stickler deps shrink, add evaluation layer to Lambda

### Step 3 (after `stickler-eval[storage]` ships):
- Notebooks use Stickler's native parquet save/load
- Reporting pipeline optionally writes consolidated parquet
- Lambda path unchanged (still reads S3 eval JSONs, cached in DynamoDB)
