# KISS: Confusion Matrix Storage — Model Change vs Metrics Dict

## Decision

How should the confusion matrix from `compare_with(include_confusion_matrix=True)` be persisted through the evaluation pipeline to S3?

| | Option A: Add to `SectionEvaluationResult` model | Option B: Embed in `metrics` dict (Recommended ✅) |
|---|---|---|
| **Change** | Add `confusion_matrix: Optional[Dict] = None` field to dataclass | Add `metrics["confusion_matrix"] = stickler_result.get("confusion_matrix", {})` |
| **Model change** | Yes — new field on `SectionEvaluationResult` | None |
| **Serialization** | Need to ensure new field is included in JSON output | Already flows through — `metrics` is `Dict[str, Any]` |
| **Downstream impact** | Any code that constructs `SectionEvaluationResult` needs updating | None — `metrics` is already a flexible dict |
| **Discoverability** | Explicit — shows up in type hints and IDE autocomplete | Implicit — hidden inside metrics dict |
| **Lines changed** | ~5 (model + transform) | ~1 (transform only) |

## Recommendation

**Use Option B (embed in metrics dict).**

The confusion matrix is a transient artifact consumed only by the bulk aggregator. It doesn't need first-class model representation. The `metrics` dict already carries `weighted_overall_score` and other derived values — adding `confusion_matrix` follows the same pattern.

## Implementation

In `_transform_stickler_result()`, before the return:

```python
# Add confusion matrix for bulk aggregation (consumed by BulkEvaluationAggregator)
metrics["confusion_matrix"] = stickler_result.get("confusion_matrix", {})
```

One line. Done.
