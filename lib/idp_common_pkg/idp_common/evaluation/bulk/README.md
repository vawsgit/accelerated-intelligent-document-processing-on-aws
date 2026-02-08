# Bulk Evaluation 📊

Aggregate evaluation metrics across multiple documents to understand extraction quality at scale.

## What It Does 🎯

- **Accumulates** per-document confusion matrices (TP/FP/FN/TN) into aggregate totals
- **Computes** field-level Precision, Recall, F1, and Accuracy across your entire document set
- **Surfaces** worst-performing fields so you know exactly where to tune prompts

## Why It Matters 🔍

Per-document metrics tell you how one doc performed. Bulk metrics tell you how your **extraction schema performs overall** — essential for production readiness.

## Quick Example 🚀

```python
from idp_common.evaluation.bulk import BulkEvaluationAggregator

aggregator = BulkEvaluationAggregator()

for doc_result in evaluation_results:
    aggregator.update(doc_result.get("confusion_matrix", {}))

metrics = aggregator.compute()
# → overall P/R/F1 + per-field breakdown
```

## Where You'll See It 👀

- **Test Studio UI** → Field-Level Metrics table (sorted by worst F1)
- **Notebooks** → `step7_bulk_evaluation.ipynb`
- **CLI** → Aggregate metrics in evaluation reports
