# Bulk Evaluator Aggregation Integration Plan — IDP Accelerator × Stickler

📌 **Original Issue**: [#179 — Bulk Evaluation Aggregation](https://github.com/aws-solutions-library-samples/accelerated-intelligent-document-processing-on-aws/issues/179)

## ⚠️ KISS Decisions Required Before Implementation

Review these decisions before building. Each links to a detailed analysis with pros/cons:

| # | Decision | Options | Recommended | Doc |
|---|----------|---------|-------------|-----|
| 1 | **Import Stickler directly or reimplement accumulation?** | Import `BulkStructuredModelEvaluator` vs custom `BulkEvaluationAggregator` | Custom aggregator (~80 lines) — Stickler's API takes StructuredModel instances (mismatch), adds 221 MB deps. Swap to Stickler native after refactor ships. | [kiss/stickler-import.md](./kiss/stickler-import.md) |
| 2 | **Retrieve confusion matrices from Athena or S3?** | Add Athena parquet column vs read eval JSONs from S3 | S3 direct — no schema migration, no backfill, cached in DynamoDB after first run | [kiss/s3-vs-athena.md](./kiss/s3-vs-athena.md) |
| 3 | **Store confusion matrix in model or metrics dict?** | New field on `SectionEvaluationResult` vs embed in existing `metrics` dict | Metrics dict — one line change, no model changes, follows existing pattern | [kiss/confusion-matrix-storage.md](./kiss/confusion-matrix-storage.md) |
| 4 | **Aggregation data source & engine?** | S3 JSON + custom, Stickler native, Athena SQL, or consolidated parquet | S3 JSON + custom aggregator — simplest, works today, cached in DynamoDB. Parquet path for notebooks later. | [kiss/aggregation-data-source.md](./kiss/aggregation-data-source.md) |

---

## Detailed Plan Documents

| Document | Description |
|----------|-------------|
| [Data Flow](./data-flow.md) | End-to-end data flow with Mermaid diagrams, current vs proposed state, confusion matrix structure |
| [Aggregator Design](./aggregator-design.md) | `BulkEvaluationAggregator` class API, input/output shapes, field path resolution, design decisions |
| [Eval Service Changes](./eval-service-changes.md) | Changes to `evaluate_section()` and `_transform_stickler_result()` with before/after code |
| [Schema & API](./schema-and-api.md) | GraphQL schema changes, `fieldLevelMetrics` JSON shape, resolver changes, query updates |
| [UI Changes](./ui-changes.md) | Test Studio wireframes, field-level metrics table, color coding, component structure |
| [Testing & Verification](./testing.md) | 3-layer test plan: unit tests, notebook verification, integration test — with code samples and reviewer checklist |
| [Future: Stickler Refactor](./future-stickler-refactor.md) | Migration path: `aggregate_from_comparisons()`, optional `stickler-eval[storage]`, swap timeline |

---

## Executive Summary

Integrate Stickler's `BulkStructuredModelEvaluator` aggregation logic into the IDP Accelerator's Test Studio metrics view. Currently, the Test Studio aggregates metrics via SQL (Athena AVG queries over per-document rows). This plan replaces that with Python-based aggregation using Stickler's confusion-matrix accumulation, yielding field-level TP/FP/FN/TN counts, derived P/R/F1/Accuracy, and weighted scoring across the full document set — surfaced in the Test Studio UI.

### What will be done

- Add a `BulkEvaluationAggregator` class in a new `idp_common.evaluation.bulk` subpackage that replicates Stickler's `BulkStructuredModelEvaluator` accumulation logic (without importing stickler directly, to avoid Lambda dependency issues)
- Modify the `test_results_resolver` Lambda to call the new aggregator instead of (or in addition to) the current Athena AVG queries
- Extend the GraphQL `TestRun` type with a `fieldLevelMetrics` field
- Update the Test Studio `TestResults.jsx` UI to display field-level metrics (P/R/F1 per field, sorted by worst-performing)
- Store per-document confusion matrix data in the reporting pipeline so it can be re-aggregated
- Create/update a notebook demonstrating bulk evaluation

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
│           ├── __init__.py                          # Export new aggregator
│           ├── bulk/                                # NEW — dedicated subpackage
│           │   ├── __init__.py                      # Package init + exports
│           │   ├── README.md                        # Feature docs (create first)
│           │   └── aggregator.py                    # BulkEvaluationAggregator class
│           └── service.py                           # Add confusion_matrix to eval output
├── nested/appsync/
│   └── src/
│       ├── api/schema.graphql                       # Add fieldLevelMetrics to TestRun
│       └── lambda/test_results_resolver/
│           ├── index.py                             # Call bulk aggregator
│           └── bulk_aggregator.py                   # NEW — standalone copy of aggregator (no deps)
├── src/
│   └── ui/src/components/test-studio/
│       └── TestResults.jsx                          # Field-level metrics UI
├── notebooks/examples/
│   └── step7_bulk_evaluation.ipynb                  # NEW — demo notebook
└── lib/idp_common_pkg/tests/unit/evaluation/
    └── test_bulk_aggregator.py                      # NEW — unit tests
```

> **Note:** `bulk_aggregator.py` exists in two places — the Lambda directory (standalone, zero deps) and `idp_common/evaluation/bulk/aggregator.py` (for notebooks/CLI). Both are identical ~80-line files. This avoids adding Lambda layers. See [kiss/stickler-import.md](./kiss/stickler-import.md) for rationale.

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

### Phase 2: Create `BulkEvaluationAggregator` in `evaluation/bulk/` subpackage

This is the core aggregation class. It replicates the accumulation logic from `BulkStructuredModelEvaluator` but operates on pre-computed confusion matrix dicts (not StructuredModel instances), since the per-document evaluation has already happened.

The new subpackage lives at `idp_common/evaluation/bulk/`, following the project's pattern of organizing sub-features into subfolders (like `agents/analytics/`, `agents/testing/`). This gives the feature a dedicated home for its README, aggregator, models, and any future additions.

File: `lib/idp_common_pkg/idp_common/evaluation/bulk/__init__.py`

```python
"""Bulk evaluation aggregation for multi-document metrics."""

from idp_common.evaluation.bulk.aggregator import BulkEvaluationAggregator

__all__ = ["BulkEvaluationAggregator"]
```

File: `lib/idp_common_pkg/idp_common/evaluation/bulk/aggregator.py`

```python
"""
Bulk evaluation aggregator for multi-document metrics.

Accumulates per-document confusion matrix results into aggregate
field-level and overall metrics. Replicates the accumulation logic
from stickler's BulkStructuredModelEvaluator without requiring
stickler as a runtime dependency.
"""

from collections import defaultdict
from typing import Any, Dict, List, Optional


class BulkEvaluationAggregator:
    """
    Accumulates confusion matrix results across documents to produce
    aggregate field-level and overall P/R/F1/Accuracy metrics.
    """

    METRIC_KEYS = ("tp", "fp", "tn", "fn", "fd", "fa")

    def __init__(self):
        self.reset()

    def reset(self):
        self._overall = defaultdict(int)
        self._fields = defaultdict(lambda: defaultdict(int))
        self._doc_count = 0
        self._errors = []

    def update(self, confusion_matrix: Dict[str, Any], doc_id: Optional[str] = None):
        """Accumulate one document's confusion matrix."""
        if not confusion_matrix:
            return
        self._doc_count += 1

        # Overall
        if "overall" in confusion_matrix:
            for k, v in confusion_matrix["overall"].items():
                if k in self.METRIC_KEYS and isinstance(v, (int, float)):
                    self._overall[k] += v

        # Fields (recursive)
        if "fields" in confusion_matrix:
            self._accumulate_fields(confusion_matrix["fields"], "")

    def _accumulate_fields(self, fields: Dict, prefix: str):
        """Recursively accumulate field metrics with dotted paths."""
        for name, data in fields.items():
            path = f"{prefix}.{name}" if prefix else name
            if not isinstance(data, dict):
                continue

            # Direct metrics
            for k in self.METRIC_KEYS:
                if k in data and isinstance(data[k], (int, float)):
                    self._fields[path][k] += data[k]

            # Hierarchical: overall + fields
            if "overall" in data:
                for k, v in data["overall"].items():
                    if k in self.METRIC_KEYS and isinstance(v, (int, float)):
                        self._fields[path][k] += v
            if "fields" in data and isinstance(data["fields"], dict):
                self._accumulate_fields(data["fields"], path)

            # List fields with nested_fields
            if "nested_fields" in data:
                for nf_name, nf_data in data["nested_fields"].items():
                    nf_path = f"{path}.{nf_name}"
                    for k, v in nf_data.items():
                        if k in self.METRIC_KEYS and isinstance(v, (int, float)):
                            self._fields[nf_path][k] += v

    def compute(self) -> Dict[str, Any]:
        """Return aggregated metrics."""
        return {
            "document_count": self._doc_count,
            "overall": self._derive(dict(self._overall)),
            "fields": {
                path: self._derive(dict(counts))
                for path, counts in sorted(self._fields.items())
            },
            "errors": self._errors,
        }

    @staticmethod
    def _derive(cm: Dict[str, int]) -> Dict[str, Any]:
        tp, fp, fn, tn = cm.get("tp", 0), cm.get("fp", 0), cm.get("fn", 0), cm.get("tn", 0)
        p = tp / (tp + fp) if (tp + fp) > 0 else 0.0
        r = tp / (tp + fn) if (tp + fn) > 0 else 0.0
        f1 = 2 * p * r / (p + r) if (p + r) > 0 else 0.0
        acc = (tp + tn) / (tp + tn + fp + fn) if (tp + tn + fp + fn) > 0 else 0.0
        return {**cm, "precision": p, "recall": r, "f1": f1, "accuracy": acc}
```

#### 2b. Export from `evaluation/__init__.py`

File: `lib/idp_common_pkg/idp_common/evaluation/__init__.py`

```python
# Bulk aggregation
from idp_common.evaluation.bulk import BulkEvaluationAggregator
```

---

### Phase 3: Integrate aggregator into test_results_resolver Lambda

File: `nested/appsync/src/lambda/test_results_resolver/index.py`
File: `nested/appsync/src/lambda/test_results_resolver/bulk_aggregator.py` ← **NEW** (standalone copy)

The current `_aggregate_test_run_metrics()` function queries Athena for AVG metrics. We need to add a parallel path that:

1. Queries Athena for document IDs in the test run
2. Reads each eval JSON from S3
3. Feeds confusion matrices into `BulkEvaluationAggregator.update()`
4. Calls `compute()` to get field-level metrics
5. Returns these alongside the existing metrics

#### 3a. Copy `bulk_aggregator.py` into Lambda directory

Copy `idp_common/evaluation/bulk/aggregator.py` → `test_results_resolver/bulk_aggregator.py`. This is a standalone ~80-line file with zero imports beyond `collections.defaultdict` and `typing`. No Lambda layers needed.

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

This is a JSON blob containing the output of `BulkEvaluationAggregator.compute()` — overall and per-field P/R/F1/Accuracy with raw TP/FP/FN/TN counts.

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

Demonstrate:
1. Loading multiple document evaluation results
2. Using `BulkEvaluationAggregator` to accumulate
3. Inspecting field-level metrics
4. Identifying worst-performing fields

This notebook also serves as the Layer 2 verification artifact — see [testing.md](./testing.md) for the full test plan including synthetic fixtures, hand-verifiable assertions, and the reviewer checklist.

#### 6b. Unit tests

File: `lib/idp_common_pkg/tests/unit/evaluation/test_bulk_aggregator.py`

See [testing.md](./testing.md) for test cases and sample code.

#### 6c. Update evaluation README and create bulk README

File: `lib/idp_common_pkg/idp_common/evaluation/README.md`

Add section pointing to the new `bulk/` subpackage for multi-document aggregation.

File: `lib/idp_common_pkg/idp_common/evaluation/bulk/README.md`

Primary documentation for the feature: what it does, how it maps to Stickler's `BulkStructuredModelEvaluator`, usage examples, and the data flow from per-document confusion matrices to aggregated field-level metrics. This file should be created first, before any code, to document intent and design decisions.

---

## KISS Review

See the [kiss/](./kiss/) directory for detailed analysis of each simplification decision:

- [kiss/stickler-import.md](./kiss/stickler-import.md) — Import Stickler directly vs custom aggregator
- [kiss/s3-vs-athena.md](./kiss/s3-vs-athena.md) — S3 direct reads vs Athena column for confusion matrix retrieval
- [kiss/confusion-matrix-storage.md](./kiss/confusion-matrix-storage.md) — Model field vs metrics dict for confusion matrix persistence
- [kiss/aggregation-data-source.md](./kiss/aggregation-data-source.md) — Data source and aggregation engine selection

---

## Code Samples

### BulkEvaluationAggregator usage pattern (Lambda context)

```python
from idp_common.evaluation.bulk import BulkEvaluationAggregator
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
import json
from idp_common.evaluation.bulk import BulkEvaluationAggregator

# Load eval results (from S3 or local)
eval_files = ["doc1_eval.json", "doc2_eval.json", ...]

aggregator = BulkEvaluationAggregator()
for f in eval_files:
    with open(f) as fh:
        result = json.load(fh)
    aggregator.update(result.get("confusion_matrix", {}), doc_id=f)

metrics = aggregator.compute()

# Display field-level metrics sorted by F1
import pandas as pd
df = pd.DataFrame([
    {"field": k, **v} for k, v in metrics["fields"].items()
]).sort_values("f1")
df[["field", "precision", "recall", "f1", "tp", "fp", "fn"]]
```
