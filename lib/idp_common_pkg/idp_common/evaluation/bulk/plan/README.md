# Bulk Evaluator Aggregation Integration Plan — IDP Accelerator × Stickler

📌 **Original Issue**: [#179 — Bulk Evaluation Aggregation](https://github.com/aws-solutions-library-samples/accelerated-intelligent-document-processing-on-aws/issues/179)

> **🔄 Updated 2026-02-12:** [Stickler PR #74](https://github.com/awslabs/stickler/pull/74) merged `aggregate_from_comparisons()` and `update_from_comparison_result()` into Stickler's `dev` branch. This resolves the API mismatch that drove KISS Decision #1. See [.working/pr-74-integration/research.md](./.working/pr-74-integration/research.md) for full analysis. Plan updated below to reflect two-tier approach: Stickler-native for `idp_common`, standalone shim for Lambda only.

## ⚠️ KISS Decisions Required Before Implementation

Review these decisions before building. Each links to a detailed analysis with pros/cons:

| # | Decision | Options | Recommended | Doc |
|---|----------|---------|-------------|-----|
| 1 | **Import Stickler directly or reimplement accumulation?** | Import `aggregate_from_comparisons()` vs custom `BulkEvaluationAggregator` | **Two-tier** — Use Stickler's `aggregate_from_comparisons()` in `idp_common` (notebooks, CLI, Docker Lambdas). Keep standalone ~80-line shim in `test_results_resolver` Lambda only (bare Zip, can't import Stickler's 221 MB deps). | [kiss/stickler-import.md](./kiss/stickler-import.md) |
| 2 | **Retrieve confusion matrices from Athena or S3?** | Add Athena parquet column vs read eval JSONs from S3 | S3 direct — no schema migration, no backfill, cached in DynamoDB after first run | [kiss/s3-vs-athena.md](./kiss/s3-vs-athena.md) |
| 3 | **Store confusion matrix in model or metrics dict?** | New field on `SectionEvaluationResult` vs embed in existing `metrics` dict | Metrics dict — one line change, no model changes, follows existing pattern | [kiss/confusion-matrix-storage.md](./kiss/confusion-matrix-storage.md) |
| 4 | **Aggregation data source & engine?** | S3 JSON + custom, Stickler native, Athena SQL, or consolidated parquet | S3 JSON + Stickler `aggregate_from_comparisons()` for `idp_common`; S3 JSON + standalone shim for Lambda. Cached in DynamoDB. | [kiss/aggregation-data-source.md](./kiss/aggregation-data-source.md) |

---

## Detailed Plan Documents

| Document | Description |
|----------|-------------|
| [Data Flow](./data-flow.md) | End-to-end data flow with Mermaid diagrams, current vs proposed state, confusion matrix structure |
| [Aggregator Design](./aggregator-design.md) | Two-tier approach: Stickler-native for `idp_common`, standalone shim API for Lambda, input/output shapes, field path resolution |
| [Eval Service Changes](./eval-service-changes.md) | Changes to `evaluate_section()` and `_transform_stickler_result()` with before/after code |
| [Schema & API](./schema-and-api.md) | GraphQL schema changes, `fieldLevelMetrics` JSON shape, resolver changes, query updates |
| [UI Changes](./ui-changes.md) | Test Studio wireframes, field-level metrics table, color coding, component structure |
| [Testing & Verification](./testing.md) | 3-layer test plan: unit tests, notebook verification, integration test — with code samples and reviewer checklist |
| [Future: Stickler Refactor](./future-stickler-refactor.md) | Migration path: `aggregate_from_comparisons()`, optional `stickler-eval[storage]`, swap timeline |

---

## Executive Summary

Integrate Stickler's `BulkStructuredModelEvaluator` aggregation logic into the IDP Accelerator's Test Studio metrics view. Currently, the Test Studio aggregates metrics via SQL (Athena AVG queries over per-document rows). This plan replaces that with Python-based aggregation using Stickler's confusion-matrix accumulation, yielding field-level TP/FP/FN/TN counts, derived P/R/F1/Accuracy, and weighted scoring across the full document set — surfaced in the Test Studio UI.

### What will be done

- Use Stickler's new `aggregate_from_comparisons()` function ([PR #74](https://github.com/awslabs/stickler/pull/74)) in `idp_common` for notebooks, CLI, and Docker Lambda contexts
- Maintain a standalone zero-dep `bulk_aggregator.py` shim in the `test_results_resolver` Lambda (bare Zip Lambda, can't import Stickler's 221 MB deps)
- Modify the `test_results_resolver` Lambda to call the aggregator instead of (or in addition to) the current Athena AVG queries
- Extend the GraphQL `TestRun` type with a `fieldLevelMetrics` field
- Update the Test Studio `TestResults.jsx` UI to display field-level metrics (P/R/F1 per field, sorted by worst-performing)
- Store per-document confusion matrix data in the reporting pipeline so it can be re-aggregated
- Create/update a notebook demonstrating bulk evaluation using Stickler's native API

### What will NOT be done

- Replace the existing per-document Stickler evaluation (`EvaluationService.evaluate_section`) — that stays as-is
- Remove the existing Athena-based cost aggregation — cost metrics remain SQL-based
- Add the full `BulkStructuredModelEvaluator` class with checkpointing/distributed merge to the accelerator — only the accumulation and compute logic
- Modify the document processing pipeline (Step Functions, Lambda orchestration)

### Files to be modified

```
accelerated-intelligent-document-processing-on-aws/
├── lib/idp_common_pkg/
│   └── idp_common/
│       └── evaluation/
│           ├── __init__.py                          # Re-export aggregate_from_comparisons from Stickler
│           ├── bulk/                                # NEW — dedicated subpackage
│           │   ├── __init__.py                      # Package init + re-exports from Stickler
│           │   └── README.md                        # Feature docs (create first)
│           └── service.py                           # Add confusion_matrix to eval output
├── nested/appsync/
│   └── src/
│       ├── api/schema.graphql                       # Add fieldLevelMetrics to TestRun
│       └── lambda/test_results_resolver/
│           ├── index.py                             # Call bulk aggregator
│           └── bulk_aggregator.py                   # NEW — standalone zero-dep shim (~80 lines)
├── src/
│   └── ui/src/components/test-studio/
│       └── TestResults.jsx                          # Field-level metrics UI
├── notebooks/examples/
│   └── step7_bulk_evaluation.ipynb                  # NEW — demo notebook (uses Stickler directly)
├── lib/idp_common_pkg/tests/unit/evaluation/
│   └── test_bulk_aggregator.py                      # NEW — unit tests + parity test vs Stickler
└── pyproject.toml                                   # Bump stickler-eval version when released
```

> **Note:** The standalone `bulk_aggregator.py` exists only in the Lambda directory. Unlike the original plan, `idp_common` no longer needs its own copy — it uses `from stickler import aggregate_from_comparisons` directly. The Lambda shim is the only custom aggregation code. See [kiss/stickler-import.md](./kiss/stickler-import.md) for rationale.

---

## Implementation Phases

### Phase 1: Ensure per-document confusion matrix data is available

The `BulkStructuredModelEvaluator.update()` calls `gt_model.compare_with(pred_model, include_confusion_matrix=True)` and accumulates the `confusion_matrix` key from the result. The IDP accelerator's `EvaluationService._transform_stickler_result()` already calls `compare_with()` but may not persist the confusion matrix. We need to ensure it does.

#### 1a. Verify `compare_with` output includes confusion_matrix

Check that `EvaluationService.evaluate_section()` passes `include_confusion_matrix=True` and that the raw confusion matrix is preserved in the evaluation results JSON stored to S3.

File: `lib/idp_common_pkg/idp_common/evaluation/service.py`

In `evaluate_section()` (around line 1336), ensure the `compare_with` call includes:
```python
result = gt_model.compare_with(
    pred_model,
    include_confusion_matrix=True,
    document_non_matches=True,
)
```

#### 1b. Persist confusion_matrix in evaluation results JSON

In `_transform_stickler_result()`, ensure the `confusion_matrix` key from the Stickler result is included in the output dict that gets saved to S3:

```python
# In the result dict being built:
"confusion_matrix": stickler_result.get("confusion_matrix", {}),
```

#### 1c. ~~Add confusion_matrix to reporting parquet schema~~ (Deferred)

> **Moved to future work.** The initial implementation reads confusion matrices from S3 eval JSONs directly (KISS decision #4). Adding a `confusion_matrix_json` column to the reporting parquet is a future enhancement for notebook-based analysis. See [future-stickler-refactor.md](./future-stickler-refactor.md).

---

### Phase 2: Wire Stickler's `aggregate_from_comparisons()` into `evaluation/bulk/` subpackage

> **Updated 2026-02-12:** [Stickler PR #74](https://github.com/awslabs/stickler/pull/74) shipped `aggregate_from_comparisons()` and `update_from_comparison_result()`. The custom `BulkEvaluationAggregator` class is no longer needed in `idp_common`. This phase now creates a thin re-export subpackage instead of reimplementing accumulation logic.

The new subpackage lives at `idp_common/evaluation/bulk/`, following the project's pattern of organizing sub-features into subfolders (like `agents/analytics/`, `agents/testing/`). This gives the feature a dedicated home for its README and any future additions.

File: `lib/idp_common_pkg/idp_common/evaluation/bulk/__init__.py`

```python
"""Bulk evaluation aggregation for multi-document metrics.

Uses Stickler's aggregate_from_comparisons() to accumulate pre-computed
compare_with() results into field-level P/R/F1/Accuracy metrics.
"""

from stickler import aggregate_from_comparisons

__all__ = ["aggregate_from_comparisons"]
```

#### 2b. Export from `evaluation/__init__.py`

File: `lib/idp_common_pkg/idp_common/evaluation/__init__.py`

```python
# Bulk aggregation (Stickler-native)
from idp_common.evaluation.bulk import aggregate_from_comparisons
```

#### 2c. Bump `stickler-eval` version

File: `pyproject.toml`

Update the `stickler-eval` pin to the release containing PR #74 (version TBD — PR merged to `dev`, awaiting release):

```toml
# When released:
"stickler-eval>=0.2.0"  # or whatever version ships PR #74
```

> **⚠️ Blocking:** PR #74 merged to Stickler's `dev` branch, not `main`. The IDP Accelerator currently pins `stickler-eval==0.1.4`. Implementation of this phase is blocked until a Stickler release includes these changes. Track via Stickler releases.

---

### Phase 3: Integrate aggregator into test_results_resolver Lambda

File: `nested/appsync/src/lambda/test_results_resolver/index.py`
File: `nested/appsync/src/lambda/test_results_resolver/bulk_aggregator.py` ← **NEW** (standalone zero-dep shim)

The current `_aggregate_test_run_metrics()` function queries Athena for AVG metrics. We need to add a parallel path that:

1. Queries Athena for document IDs in the test run
2. Reads each eval JSON from S3
3. Feeds confusion matrices into the aggregator
4. Returns field-level metrics alongside the existing metrics

#### 3a. Create standalone `bulk_aggregator.py` in Lambda directory

This is a zero-dep reimplementation of Stickler's accumulation logic (~80 lines). It exists because the `test_results_resolver` is a bare Zip Lambda with no access to Stickler's 221 MB dependency chain.

The shim should match the behavior of Stickler's `update_from_comparison_result()` — specifically:
- Validate that input contains a `"confusion_matrix"` key
- Accumulate TP/FP/FN/TN counts per field using dot-path flattening
- Derive P/R/F1/Accuracy from accumulated counts

> **Parity requirement:** A test must verify that the Lambda shim produces identical output to `stickler.aggregate_from_comparisons()` for the same input. See [testing.md](./testing.md).

```python
# In index.py
from bulk_aggregator import BulkEvaluationAggregator
```

#### 3b. Add field-level metrics retrieval

#### 3b. Add field-level metrics retrieval

```python
def _get_field_level_metrics(test_run_id):
    """Read per-document eval JSONs from S3, extract confusion matrices, aggregate."""
    output_bucket = os.environ.get('OUTPUT_BUCKET')
    if not output_bucket:
        return {}

    # 1. Get document keys from Athena (reuse existing query pattern)
    doc_keys = _get_document_keys_for_test_run(test_run_id)
    if not doc_keys:
        return {}

    # 2. Read eval JSONs from S3, feed confusion matrices into aggregator
    from bulk_aggregator import BulkEvaluationAggregator
    aggregator = BulkEvaluationAggregator()

    for doc_key in doc_keys:
        try:
            eval_key = f"{doc_key}/evaluation/results.json"
            response = s3_client.get_object(Bucket=output_bucket, Key=eval_key)
            eval_result = json.loads(response['Body'].read())
            for section in eval_result.get('section_results', []):
                cm = section.get('metrics', {}).get('confusion_matrix', {})
                if cm:
                    aggregator.update(cm, doc_id=doc_key)
        except Exception:
            continue

    return aggregator.compute()
```

#### 3c. Wire into `_aggregate_test_run_metrics`

Add `field_level_metrics` to the returned dict:

```python
def _aggregate_test_run_metrics(test_run_id):
    evaluation_metrics = _get_evaluation_metrics_from_athena(test_run_id)
    cost_data = _get_cost_data_from_athena(test_run_id)
    field_level_metrics = _get_field_level_metrics(test_run_id)  # ← NEW

    return {
        # ... existing keys ...
        'field_level_metrics': field_level_metrics,                # ← NEW
    }
```

#### 3d. Include in cached and returned results

Update `get_test_results()` and `handle_cache_update_request()` to include `fieldLevelMetrics` in the response and cache.

---

### Phase 4: Extend GraphQL schema

File: `nested/appsync/src/api/schema.graphql`

Add `fieldLevelMetrics` to the `TestRun` type:

```graphql
type TestRun @aws_cognito_user_pools @aws_iam {
  # ... existing fields ...
  fieldLevelMetrics: AWSJSON
}
```

This is a JSON blob containing the output of the aggregator — overall and per-field P/R/F1/Accuracy with raw TP/FP/FN/TN counts. In `idp_common` contexts this comes from Stickler's `ProcessEvaluation`; in the Lambda it comes from the standalone shim's `compute()` method.

---

### Phase 5: Update Test Studio UI

File: `src/ui/src/components/test-studio/TestResults.jsx`

#### 5a. Add field-level metrics table

Add a new `ExpandableSection` (or a primary section) titled "Field-Level Metrics" that renders a sortable table with columns:

| Field | Precision | Recall | F1 | Accuracy | TP | FP | FN |
|-------|-----------|--------|----|----------|----|----|----|

Sorted by F1 ascending (worst-performing fields first) so users can immediately see which fields need prompt tuning.

#### 5b. Parse the new `fieldLevelMetrics` from GraphQL response

In the existing `getTestRun` query result handling, parse `fieldLevelMetrics` (it comes as AWSJSON string, needs `JSON.parse()`).

#### 5c. Visual indicators

- Color-code F1 scores: red (<0.5), yellow (0.5-0.8), green (>0.8)
- Show overall aggregate metrics (total TP/FP/FN across all fields) in the existing summary section
- Add a bar chart showing F1 per field (using existing recharts dependency)

#### 5d. Update GraphQL query

File: `src/ui/src/graphql/queries/getTestResults.js`

Add `fieldLevelMetrics` to the query:

```javascript
const GET_TEST_RUN = `
  query GetTestRun($testRunId: String!) {
    getTestRun(testRunId: $testRunId) {
      # ... existing fields ...
      fieldLevelMetrics
    }
  }
`;
```

---

### Phase 6: Notebook, testing, and documentation

#### 6a. Create bulk evaluation notebook

File: `notebooks/examples/step7_bulk_evaluation.ipynb`

Demonstrate using Stickler's native `aggregate_from_comparisons()`:
1. Loading multiple document evaluation results
2. Calling `aggregate_from_comparisons()` to get field-level metrics
3. Inspecting the `ProcessEvaluation` result
4. Identifying worst-performing fields

```python
from stickler import aggregate_from_comparisons

# comparison_results = [list of compare_with() result dicts loaded from S3]
result = aggregate_from_comparisons(comparison_results)
result.pretty_print_metrics()
```

This notebook also serves as the Layer 2 verification artifact — see [testing.md](./testing.md) for the full test plan including synthetic fixtures, hand-verifiable assertions, and the reviewer checklist.

#### 6b. Unit tests

File: `lib/idp_common_pkg/tests/unit/evaluation/test_bulk_aggregator.py`

Tests must include a **parity test** that verifies the Lambda's standalone `bulk_aggregator.py` produces identical output to `stickler.aggregate_from_comparisons()` for the same input. See [testing.md](./testing.md) for test cases and sample code.

#### 6c. Update evaluation README and create bulk README

File: `lib/idp_common_pkg/idp_common/evaluation/README.md`

Add section pointing to the new `bulk/` subpackage for multi-document aggregation.

File: `lib/idp_common_pkg/idp_common/evaluation/bulk/README.md`

Primary documentation for the feature: what it does, how it uses Stickler's `aggregate_from_comparisons()`, usage examples, and the data flow from per-document confusion matrices to aggregated field-level metrics. This file should be created first, before any code, to document intent and design decisions.

---

## KISS Review

See the [kiss/](./kiss/) directory for detailed analysis of each simplification decision:

- [kiss/stickler-import.md](./kiss/stickler-import.md) — Import Stickler directly vs custom aggregator
- [kiss/s3-vs-athena.md](./kiss/s3-vs-athena.md) — S3 direct reads vs Athena column for confusion matrix retrieval
- [kiss/confusion-matrix-storage.md](./kiss/confusion-matrix-storage.md) — Model field vs metrics dict for confusion matrix persistence
- [kiss/aggregation-data-source.md](./kiss/aggregation-data-source.md) — Data source and aggregation engine selection

---

## Code Samples

### Aggregation usage pattern — `idp_common` / notebooks (Stickler-native)

```python
from stickler import aggregate_from_comparisons
import json

# Load pre-computed compare_with() results (from S3 or local)
comparison_results = []
for f in eval_files:
    with open(f) as fh:
        result = json.load(fh)
    # Each result is a compare_with() output dict with "confusion_matrix" key
    comparison_results.append(result)

# Aggregate — returns ProcessEvaluation
process_eval = aggregate_from_comparisons(comparison_results)

# Access metrics
print(f"Documents: {process_eval.document_count}")
print(f"Overall: {process_eval.metrics}")
print(f"Fields: {process_eval.field_metrics}")
process_eval.pretty_print_metrics()
```

### Aggregation usage pattern — Lambda (standalone shim)

```python
from bulk_aggregator import BulkEvaluationAggregator
import json

aggregator = BulkEvaluationAggregator()

# For each document in the test run
for doc_eval_result in document_eval_results:
    cm = doc_eval_result.get("confusion_matrix", {})
    aggregator.update(cm, doc_id=doc_eval_result.get("document_id"))

# Get aggregated metrics
result = aggregator.compute()
# result = {
#   "document_count": 75,
#   "overall": {"tp": 450, "fp": 12, "fn": 8, "precision": 0.974, "recall": 0.982, "f1": 0.978, "accuracy": 0.957},
#   "fields": {
#     "invoice_id": {"tp": 75, "fp": 0, "fn": 0, "precision": 1.0, "recall": 1.0, "f1": 1.0, "accuracy": 1.0},
#     "customer_name": {"tp": 68, "fp": 3, "fn": 4, "precision": 0.957, "recall": 0.944, "f1": 0.951, "accuracy": 0.907},
#     "line_items.description": {"tp": 180, "fp": 5, "fn": 2, ...},
#   }
# }
```

### TestResults.jsx field-level metrics table (React)

```jsx
const FieldLevelMetrics = ({ fieldLevelMetrics }) => {
  if (!fieldLevelMetrics?.fields) return null;

  const items = Object.entries(fieldLevelMetrics.fields)
    .map(([field, metrics]) => ({ field, ...metrics }))
    .filter(item => (item.tp + item.fp + item.fn) > 0)
    .sort((a, b) => a.f1 - b.f1); // Worst first

  return (
    <Table
      header={<Header variant="h3">Field-Level Metrics (Aggregated)</Header>}
      items={items}
      columnDefinitions={[
        { id: 'field', header: 'Field', cell: item => item.field },
        { id: 'f1', header: 'F1', cell: item => item.f1.toFixed(3) },
        { id: 'precision', header: 'Precision', cell: item => item.precision.toFixed(3) },
        { id: 'recall', header: 'Recall', cell: item => item.recall.toFixed(3) },
        { id: 'tp', header: 'TP', cell: item => item.tp },
        { id: 'fp', header: 'FP', cell: item => item.fp },
        { id: 'fn', header: 'FN', cell: item => item.fn },
      ]}
      sortingColumn={{ id: 'f1' }}
      sortingDescending={false}
    />
  );
};
```

### Notebook cell — loading and aggregating

```python
from stickler import aggregate_from_comparisons
import json

# Load eval results (from S3 or local)
eval_files = ["doc1_eval.json", "doc2_eval.json", ...]

comparison_results = []
for f in eval_files:
    with open(f) as fh:
        comparison_results.append(json.load(fh))

result = aggregate_from_comparisons(comparison_results)

# Display field-level metrics sorted by F1
import pandas as pd
df = pd.DataFrame([
    {"field": k, **v} for k, v in result.field_metrics.items()
]).sort_values("f1")
df[["field", "precision", "recall", "f1", "tp", "fp", "fn"]]
```
