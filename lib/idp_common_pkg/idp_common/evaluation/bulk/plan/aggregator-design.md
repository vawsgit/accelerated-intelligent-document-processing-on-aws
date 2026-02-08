# Aggregator Class: Design & API

## Class: `BulkEvaluationAggregator`

Location: `lib/idp_common_pkg/idp_common/evaluation/bulk/aggregator.py`

## Public API

```python
class BulkEvaluationAggregator:
    """Accumulates per-document confusion matrices into aggregate field-level metrics."""

    def __init__(self) -> None: ...
    def reset(self) -> None: ...
    def update(self, confusion_matrix: dict, doc_id: str | None = None) -> None: ...
    def compute(self) -> dict: ...
```

| Method | Purpose |
|--------|---------|
| `__init__()` | Initialize empty accumulators |
| `reset()` | Clear all accumulated state |
| `update(cm, doc_id)` | Add one document's confusion matrix to the running totals |
| `compute()` | Return aggregated metrics dict |

## Metric Keys Tracked

| Key | Meaning |
|-----|---------|
| `tp` | True Positive — field present in both expected and actual, values match |
| `fp` | False Positive — field present in actual but not expected, or value wrong |
| `fn` | False Negative — field present in expected but missing/empty in actual |
| `tn` | True Negative — field absent in both expected and actual |
| `fd` | False Discovery (Stickler-specific) |
| `fa` | False Alarm (Stickler-specific) |

## Derived Metrics

| Metric | Formula |
|--------|---------|
| Precision | `tp / (tp + fp)` |
| Recall | `tp / (tp + fn)` |
| F1 | `2 * P * R / (P + R)` |
| Accuracy | `(tp + tn) / (tp + tn + fp + fn)` |

All derived metrics default to `0.0` when the denominator is zero.

## Input: Confusion Matrix Shape

The `update()` method accepts the `confusion_matrix` dict from Stickler's `compare_with(include_confusion_matrix=True)`:

```python
# Flat field
{"fields": {"invoice_id": {"tp": 1, "fp": 0, "fn": 0, "tn": 0}}}

# Nested/list field (has overall + nested_fields)
{"fields": {"line_items": {
    "overall": {"tp": 3, "fp": 1, "fn": 0, "tn": 0},
    "nested_fields": {
        "description": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
        "amount": {"tp": 1, "fp": 1, "fn": 0, "tn": 0}
    }
}}}

# Hierarchical (has overall + fields)
{"fields": {"address": {
    "overall": {"tp": 2, "fp": 0, "fn": 0, "tn": 0},
    "fields": {
        "street": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
        "city": {"tp": 1, "fp": 0, "fn": 0, "tn": 0}
    }
}}}
```

## Output: `compute()` Return Shape

```python
{
    "document_count": 75,
    "overall": {
        "tp": 450, "fp": 12, "fn": 8, "tn": 5,
        "precision": 0.974, "recall": 0.982, "f1": 0.978, "accuracy": 0.957
    },
    "fields": {
        "invoice_id":              {"tp": 75, "fp": 0, "fn": 0, "tn": 0, "precision": 1.0, ...},
        "customer_name":           {"tp": 68, "fp": 3, "fn": 4, "tn": 0, ...},
        "line_items":              {"tp": 210, "fp": 6, "fn": 3, "tn": 0, ...},
        "line_items.description":  {"tp": 70, "fp": 2, "fn": 1, "tn": 0, ...},
        "line_items.amount":       {"tp": 70, "fp": 3, "fn": 1, "tn": 0, ...},
        "line_items.quantity":     {"tp": 70, "fp": 1, "fn": 1, "tn": 0, ...},
    },
    "errors": []
}
```

Note: Fields are **dot-path flattened** — `line_items.description` represents the `description` nested field within `line_items`. Both the parent (`line_items`) and children appear in the output.

## Field Path Resolution

```mermaid
graph TD
    A[confusion_matrix.fields] --> B{Has direct tp/fp/fn?}
    B -->|Yes| C["Accumulate at 'field_name'"]
    B -->|No| D{Has 'overall' key?}
    D -->|Yes| E["Accumulate at 'field_name'"]
    D -->|No| F[Skip]
    A --> G{Has 'fields' sub-dict?}
    G -->|Yes| H["Recurse with prefix 'field_name.'"]
    A --> I{Has 'nested_fields'?}
    I -->|Yes| J["Accumulate each at 'field_name.nested_name'"]
```

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| No Stickler import | Avoids adding stickler as Lambda dependency; operates on plain dicts |
| Micro-averaging | Each field instance counts equally — standard for NER/extraction tasks |
| Dot-path flattening | Enables flat table display in UI while preserving hierarchy info |
| `errors` list | Captures malformed confusion matrices without failing the whole aggregation |
| `reset()` method | Enables reuse of aggregator instance across test runs |
