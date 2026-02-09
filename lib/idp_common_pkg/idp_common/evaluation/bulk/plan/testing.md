# Testing & Verification Plan

This document details how to verify the bulk evaluation feature works end-to-end. It is designed for a **human reviewer** approving the implementation PR.

---

## Test Layers

| Layer | What | How | Who |
|-------|------|-----|-----|
| 1. Unit tests | `BulkEvaluationAggregator` logic | `pytest` — automated | CI |
| 2. Notebook verification | Aggregator with real confusion matrices | Jupyter notebook — manual | Reviewer |
| 3. Integration test | Full pipeline: upload → evaluate → aggregate → UI | IDP CLI + Test Studio UI — manual | Reviewer |

---

## Layer 1: Unit Tests (Automated)

File: `lib/idp_common_pkg/tests/unit/evaluation/test_bulk_aggregator.py`

These run in CI. The reviewer should confirm they pass and cover the cases below.

### Test Cases

| Test | Input | Expected Output |
|------|-------|-----------------|
| Empty input | `aggregator.update({})` × 3, then `compute()` | `document_count: 0`, empty fields |
| Single flat doc | 1 doc with 3 flat fields (invoice_id, date, total) | Field-level P/R/F1 match hand-calculated values |
| Multiple flat docs | 3 docs with same fields, varying TP/FP/FN | Counts sum correctly, derived metrics are micro-averaged |
| Nested fields | Doc with `line_items.description`, `line_items.amount` via `nested_fields` | Dot-path fields appear in output |
| Hierarchical fields | Doc with `address.overall` + `address.fields.street/city` | Both parent and child paths accumulated |
| Mixed docs | 5 docs: 3 with confusion_matrix, 1 empty `{}`, 1 `None` | `document_count: 3`, empty/None skipped |
| Reset | `update()` × 3, `reset()`, `update()` × 1 | Only last doc counted |
| Zero denominators | All FN, no TP or FP | `precision: 0.0`, `recall: 0.0`, `f1: 0.0` |

### Sample Test Code

```python
import pytest
from idp_common.evaluation.bulk import BulkEvaluationAggregator


class TestBulkEvaluationAggregator:

    def test_single_document_flat_fields(self):
        agg = BulkEvaluationAggregator()
        agg.update({
            "overall": {"tp": 2, "fp": 1, "fn": 0, "tn": 0},
            "fields": {
                "invoice_id": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                "total": {"tp": 1, "fp": 1, "fn": 0, "tn": 0},
            }
        })
        result = agg.compute()
        assert result["document_count"] == 1
        assert result["overall"]["tp"] == 2
        assert result["overall"]["fp"] == 1
        assert result["fields"]["invoice_id"]["precision"] == 1.0
        assert result["fields"]["total"]["precision"] == pytest.approx(0.5)

    def test_multiple_documents_accumulate(self):
        agg = BulkEvaluationAggregator()
        for _ in range(3):
            agg.update({
                "overall": {"tp": 1, "fp": 0, "fn": 1, "tn": 0},
                "fields": {
                    "name": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                    "date": {"tp": 0, "fp": 0, "fn": 1, "tn": 0},
                }
            })
        result = agg.compute()
        assert result["document_count"] == 3
        assert result["overall"]["tp"] == 3
        assert result["overall"]["fn"] == 3
        assert result["fields"]["name"]["tp"] == 3
        assert result["fields"]["name"]["f1"] == 1.0
        assert result["fields"]["date"]["fn"] == 3
        assert result["fields"]["date"]["recall"] == 0.0

    def test_nested_fields(self):
        agg = BulkEvaluationAggregator()
        agg.update({
            "overall": {"tp": 3, "fp": 0, "fn": 0, "tn": 0},
            "fields": {
                "line_items": {
                    "overall": {"tp": 3, "fp": 0, "fn": 0, "tn": 0},
                    "nested_fields": {
                        "description": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                        "amount": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                        "quantity": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                    }
                }
            }
        })
        result = agg.compute()
        assert "line_items" in result["fields"]
        assert "line_items.description" in result["fields"]
        assert "line_items.amount" in result["fields"]

    def test_empty_and_none_skipped(self):
        agg = BulkEvaluationAggregator()
        agg.update({"overall": {"tp": 1, "fp": 0, "fn": 0, "tn": 0}, "fields": {}})
        agg.update({})
        agg.update(None)
        assert agg.compute()["document_count"] == 1
```

### Running

```bash
cd lib/idp_common_pkg
pytest tests/unit/evaluation/test_bulk_aggregator.py -v
```

### Sync Check

The aggregator exists in two places. Verify they're identical:

```bash
diff lib/idp_common_pkg/idp_common/evaluation/bulk/aggregator.py \
     nested/appsync/src/lambda/test_results_resolver/bulk_aggregator.py
```

This should produce no output. Add this check to the reviewer checklist.

---

## Layer 2: Notebook Verification (Manual)

File: `notebooks/examples/step7_bulk_evaluation.ipynb`

This notebook lets the reviewer run the aggregator locally against synthetic and/or real evaluation data without needing a deployed stack.

### Prerequisites

```bash
cd lib/idp_common_pkg && pip install -e ".[evaluation]"
```

### Notebook Cells

#### Cell 1: Setup

```python
import json
from idp_common.evaluation.bulk import BulkEvaluationAggregator
import pandas as pd

print("BulkEvaluationAggregator imported successfully ✅")
```

#### Cell 2: Synthetic test data (3 invoice documents)

This is the core verification fixture. The reviewer can hand-verify the math.

```python
# 3 synthetic documents with known confusion matrices
synthetic_docs = [
    {
        "doc_id": "invoice_001",
        "confusion_matrix": {
            "overall": {"tp": 4, "fp": 1, "fn": 0, "tn": 0},
            "fields": {
                "invoice_number": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                "invoice_date": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                "total_amount": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                "vendor_name": {"tp": 1, "fp": 1, "fn": 0, "tn": 0},
            }
        }
    },
    {
        "doc_id": "invoice_002",
        "confusion_matrix": {
            "overall": {"tp": 3, "fp": 0, "fn": 1, "tn": 0},
            "fields": {
                "invoice_number": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                "invoice_date": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                "total_amount": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                "vendor_name": {"tp": 0, "fp": 0, "fn": 1, "tn": 0},
            }
        }
    },
    {
        "doc_id": "invoice_003",
        "confusion_matrix": {
            "overall": {"tp": 2, "fp": 1, "fn": 1, "tn": 0},
            "fields": {
                "invoice_number": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                "invoice_date": {"tp": 0, "fp": 0, "fn": 1, "tn": 0},
                "total_amount": {"tp": 1, "fp": 0, "fn": 0, "tn": 0},
                "vendor_name": {"tp": 0, "fp": 1, "fn": 0, "tn": 0},
            }
        }
    },
]
print(f"Created {len(synthetic_docs)} synthetic documents")
```

#### Cell 3: Run aggregation

```python
aggregator = BulkEvaluationAggregator()
for doc in synthetic_docs:
    aggregator.update(doc["confusion_matrix"], doc_id=doc["doc_id"])

metrics = aggregator.compute()
print(f"Documents aggregated: {metrics['document_count']}")
print(f"Overall: P={metrics['overall']['precision']:.3f} R={metrics['overall']['recall']:.3f} F1={metrics['overall']['f1']:.3f}")
```

#### Cell 4: Display as table (sorted by worst F1)

```python
df = pd.DataFrame([
    {"field": k, **v} for k, v in metrics["fields"].items()
]).sort_values("f1")

# Expected results for hand-verification:
# invoice_number: TP=3, FP=0, FN=0 → P=1.000, R=1.000, F1=1.000
# total_amount:   TP=3, FP=0, FN=0 → P=1.000, R=1.000, F1=1.000
# invoice_date:   TP=2, FP=0, FN=1 → P=1.000, R=0.667, F1=0.800
# vendor_name:    TP=1, FP=2, FN=1 → P=0.333, R=0.500, F1=0.400

display(df[["field", "precision", "recall", "f1", "tp", "fp", "fn"]])
```

#### Cell 5: Verify against hand-calculated expected values

```python
# Automated assertions the reviewer can run
fields = metrics["fields"]

assert fields["invoice_number"]["tp"] == 3
assert fields["invoice_number"]["f1"] == 1.0

assert fields["vendor_name"]["tp"] == 1
assert fields["vendor_name"]["fp"] == 2
assert fields["vendor_name"]["fn"] == 1
assert abs(fields["vendor_name"]["precision"] - 1/3) < 0.001
assert abs(fields["vendor_name"]["f1"] - 0.4) < 0.001

assert fields["invoice_date"]["fn"] == 1
assert abs(fields["invoice_date"]["recall"] - 2/3) < 0.001

print("All assertions passed ✅")
```

#### Cell 6 (Optional): Load real eval results from S3

```python
# Only works with a deployed stack
import boto3

STACK_NAME = "IDP"  # ← reviewer sets this
OUTPUT_BUCKET = "..."  # ← from CloudFormation outputs
TEST_RUN_PREFIX = "..."  # ← from a completed test run

s3 = boto3.client("s3")
aggregator = BulkEvaluationAggregator()

# List eval result files
paginator = s3.get_paginator("list_objects_v2")
for page in paginator.paginate(Bucket=OUTPUT_BUCKET, Prefix=TEST_RUN_PREFIX):
    for obj in page.get("Contents", []):
        if obj["Key"].endswith("/evaluation/results.json"):
            response = s3.get_object(Bucket=OUTPUT_BUCKET, Key=obj["Key"])
            eval_result = json.loads(response["Body"].read())
            # Extract confusion matrix from section metrics
            for section in eval_result.get("section_results", []):
                cm = section.get("metrics", {}).get("confusion_matrix", {})
                if cm:
                    aggregator.update(cm, doc_id=eval_result.get("document_id"))

result = aggregator.compute()
print(f"Aggregated {result['document_count']} documents from S3")

df = pd.DataFrame([
    {"field": k, **v} for k, v in result["fields"].items()
]).sort_values("f1")
display(df[["field", "precision", "recall", "f1", "tp", "fp", "fn"]])
```

### What the Reviewer Should Verify

| Check | How |
|-------|-----|
| Aggregator imports cleanly | Cell 1 runs without error |
| Counts sum correctly | Cell 4 table matches hand-calculated values in comments |
| Assertions pass | Cell 5 prints "All assertions passed ✅" |
| Nested fields work | If testing with real data that has list fields, dot-paths appear |
| Empty docs handled | Add `aggregator.update({})` before Cell 3 — count should not change |

---

## Layer 3: Integration Test (Manual, Deployed Stack)

This verifies the full pipeline: document upload → evaluation → bulk aggregation → Test Studio UI.

### Prerequisites

- Deployed IDP stack (Pattern 2 recommended — has evaluation built in)
- IDP CLI installed: `cd lib/idp_cli_pkg && pip install -e .`
- Bedrock model access enabled (Claude 3.x/4.x, Nova models)
- A test set with baselines configured

### Step 1: Run a test set with evaluation

```bash
idp-cli run-inference \
    --stack-name IDP \
    --test-set fcc-example-test \
    --number-of-files 5 \
    --context "Bulk eval verification" \
    --monitor
```

Wait for all documents to complete (status: COMPLETED).

### Step 2: Verify confusion matrix in eval results JSON

Pick one completed document and check its eval results in S3:

```bash
# Get the output bucket from stack outputs
OUTPUT_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name IDP \
    --query "Stacks[0].Outputs[?OutputKey=='OutputBucketName'].OutputValue" \
    --output text)

# List eval results for the test run
aws s3 ls s3://$OUTPUT_BUCKET/ --recursive | grep "evaluation/results.json" | head -3

# Download one and check for confusion_matrix
aws s3 cp s3://$OUTPUT_BUCKET/<path>/evaluation/results.json /tmp/eval_result.json
python3 -c "
import json
r = json.load(open('/tmp/eval_result.json'))
for s in r.get('section_results', []):
    cm = s.get('metrics', {}).get('confusion_matrix')
    print(f\"Section {s['section_id']}: confusion_matrix={'present' if cm else 'MISSING'}\")
    if cm:
        print(f\"  overall: {cm.get('overall', {})}\")
        print(f\"  fields: {list(cm.get('fields', {}).keys())}\")
"
```

**Expected**: Each section's `metrics` dict contains a `confusion_matrix` key with `overall` and `fields`.

**If missing**: Phase 1 (`include_confusion_matrix=True` in `compare_with()`) was not implemented correctly.

### Step 3: Verify field-level metrics in Test Studio UI

1. Open the Test Studio UI (URL from CloudFormation outputs)
2. Navigate to the test run from Step 1
3. Look for the **"Field-Level Metrics (Aggregated)"** section

**Expected**:
- A table showing each extraction field with P/R/F1/TP/FP/FN columns
- Sorted by F1 ascending (worst fields first)
- Color-coded F1 badges (red < 0.5, yellow 0.5-0.8, green > 0.8)
- An overall summary showing aggregate P/R/F1

**If missing**: Check that `fieldLevelMetrics` is in the GraphQL query response (browser dev tools → Network → look for `getTestRun` query).

### Step 4: Verify GraphQL response

In browser dev tools (Network tab), find the `getTestRun` GraphQL request and check the response:

```json
{
  "data": {
    "getTestRun": {
      "fieldLevelMetrics": "{\"document_count\":5,\"overall\":{...},\"fields\":{...}}"
    }
  }
}
```

**Expected**: `fieldLevelMetrics` is a non-null JSON string. Parse it and verify `document_count` matches the number of successfully evaluated documents.

### Step 5: Cross-check notebook vs UI

Run the notebook (Layer 2, Cell 6) against the same test run's S3 data. The field-level metrics from the notebook should match what the UI displays.

| Field | Notebook F1 | UI F1 | Match? |
|-------|-------------|-------|--------|
| field_a | ... | ... | ✅/❌ |
| field_b | ... | ... | ✅/❌ |

---

## Test Data Requirements

| Data | Source | Purpose |
|------|--------|---------|
| Synthetic confusion matrices | Hardcoded in unit tests + notebook | Verify aggregation math |
| `samples/lending_package.pdf` | Already in repo | Integration test input (Pattern 2) |
| Test set with baselines | `fcc-example-test` (pre-configured) | End-to-end evaluation pipeline |

No new test data files need to be created. The synthetic fixtures in the unit tests and notebook are sufficient for verifying aggregation logic. Real evaluation data comes from running the existing test set.

---

## Reviewer Checklist

```
□ Unit tests pass: pytest tests/unit/evaluation/test_bulk_aggregator.py -v
□ Aggregator files are identical: diff aggregator.py bulk_aggregator.py (no output)
□ Notebook Cell 5 assertions pass (synthetic data)
□ Notebook Cell 6 loads real S3 data (if stack available)
□ Eval results JSON contains confusion_matrix in metrics
□ Test Studio UI shows Field-Level Metrics section
□ UI field metrics match notebook output for same test run
□ GraphQL response includes non-null fieldLevelMetrics
□ Existing metrics (overallAccuracy, costBreakdown, etc.) still work
□ Test runs without baselines gracefully show no field-level metrics
```
