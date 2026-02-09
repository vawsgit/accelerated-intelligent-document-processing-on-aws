# KISS Alternative: Direct Stickler Import

## Overview

This document evaluates the alternative approach of directly importing Stickler's `BulkStructuredModelEvaluator` class in the `test_results_resolver` Lambda instead of reimplementing the accumulation logic in a custom `BulkEvaluationAggregator` class.

---

## Key Finding: Stickler Is Already an IDP Dependency

Stickler (`stickler-eval==0.1.4`) is **already a core dependency** of the IDP Accelerator:

| Where | How |
|-------|-----|
| `pyproject.toml` → `[evaluation]` extra | `stickler-eval==0.1.4` |
| `pyproject.toml` → `[test]` extra | `stickler-eval==0.1.4` |
| `pyproject.toml` → `[all]` extra | `stickler-eval==0.1.4` |
| `evaluation/service.py` | `from stickler import StructuredModel` |
| `evaluation/llm_comparator.py` | `from stickler.structured_object_evaluator.models.comparator_registry import ...` |
| `evaluation/stickler_mapper.py` | Maps IDP config → Stickler config format |
| `evaluation/stickler_version.py` | Tracks Stickler version/commit |
| Evaluation Lambda (all patterns) | Docker image with `idp_common[evaluation,docs_service]` |

The class we want already exists at: `stickler.structured_object_evaluator.bulk_structured_model_evaluator.BulkStructuredModelEvaluator`

---

## The Packaging Problem

The `test_results_resolver` Lambda is where bulk aggregation needs to run. Here's its current packaging:

```
TestResultsResolverFunction:
  PackageType: Zip              # ← NOT Docker
  CodeUri: ./src/lambda/test_results_resolver
  Layers: (none)                # ← NO Lambda layers
  Runtime: python3.12
  MemorySize: 512
  Timeout: 300
```

It's a **bare Zip Lambda** with a single `index.py` file and **no dependencies** — it only uses `boto3` (provided by the Lambda runtime). It has no access to `idp_common` or `stickler`.

Compare with the Evaluation Lambda (where Stickler currently runs):

```
EvaluationFunction:
  PackageType: Image            # ← Docker image
  ImageUri: ...evaluation-function-${ImageVersion}
  requirements.txt: ../../lib/idp_common_pkg[evaluation,docs_service]
  MemorySize: 1024
  Timeout: 900
```

---

## What It Would Take to Import Stickler Directly

### Option A: Add `IDPCommonBaseLayer` + evaluation deps to test_results_resolver

```yaml
# nested/appsync/template.yaml
TestResultsResolverFunction:
  Properties:
    Layers:
      - !Ref IDPCommonBaseLayerArn    # ← Add base layer
    # But base layer doesn't include evaluation deps...
```

The existing Lambda layers are:

| Layer | Contents | Includes Stickler? |
|-------|----------|-------------------|
| `IDPCommonBaseLayer` | core + docs_service + image | ❌ No |
| `IDPCommonReportingLayer` | core + reporting | ❌ No |
| `IDPCommonAgentsLayer` | core + agents | ❌ No |

**None of the existing layers include Stickler.** We'd need to either:

1. Create a **new** `IDPCommonEvaluationLayer` (core + evaluation deps), or
2. Add evaluation deps to the base layer (bloats all Lambdas that use it), or
3. Convert `test_results_resolver` to a Docker image Lambda

### Option B: Create a new `IDPCommonEvaluationLayer`

```yaml
# template.yaml
IDPCommonEvaluationLayer:
  Type: AWS::Lambda::LayerVersion
  Properties:
    Description: "IDP Common evaluation layer (core + evaluation)"
    Content:
      S3Bucket: ...
      S3Key: ".../layers/<IDP_COMMON_EVALUATION_LAYER_ZIP>"
    CompatibleRuntimes:
      - python3.12
```

Build config addition in `publish.py`:

```python
layers_config = {
    "base": ["docs_service", "image"],
    "reporting": ["reporting"],
    "agents": ["agents"],
    "evaluation": ["evaluation"],    # ← NEW
}
```

### Option C: Convert test_results_resolver to Docker image

```yaml
TestResultsResolverFunction:
  PackageType: Image
  ImageUri: !Sub "${ECRRepository.RepositoryUri}:test-results-resolver-${ImageVersion}"
```

With a `requirements.txt`:
```
../../lib/idp_common_pkg[evaluation]
```

---

## Size Analysis

### Stickler + Transitive Dependencies

| Package | Size | Required by Stickler? |
|---------|------|-----------------------|
| `scipy` | **97 MB** | ✅ Direct dependency |
| `pandas` | **70 MB** | ✅ Direct dependency |
| `numpy` | **33 MB** | ✅ Direct dependency |
| `rapidfuzz` | 5.0 MB | ✅ Direct dependency |
| `pydantic` + `pydantic_core` | 8.3 MB | ✅ Direct dependency |
| `psutil` | 848 KB | ✅ Direct dependency |
| `munkres` | ~100 KB | ✅ Direct dependency |
| `jsonschema` + specs | 1.2 MB | ✅ Direct dependency |
| `stickler` itself | 1.4 MB | — |
| **Total** | **~221 MB** | |

### Lambda Layer Size Limits

| Limit | Value | Stickler Fits? |
|-------|-------|----------------|
| Single layer (zipped) | 50 MB | ❌ Unlikely (221 MB unzipped) |
| All layers combined (unzipped) | 250 MB | ⚠️ Tight — leaves ~29 MB for other layers |
| Docker image | 10 GB | ✅ Easily |

A Lambda layer with Stickler + deps would likely **exceed the 50 MB zipped limit** due to scipy (97 MB) and pandas (70 MB). This means:

- **Option A (layer)**: Probably won't work without stripping scipy/pandas
- **Option B (new evaluation layer)**: Same problem
- **Option C (Docker image)**: Works, but changes the Lambda packaging model

### What BulkStructuredModelEvaluator Actually Needs

Looking at the source, `BulkStructuredModelEvaluator` imports:

```python
from stickler.structured_object_evaluator.models.structured_model import StructuredModel
from stickler.utils.process_evaluation import ProcessEvaluation
```

It needs `StructuredModel` because its `update()` method takes `(gt_model: StructuredModel, pred_model: StructuredModel)` — it calls `gt_model.compare_with(pred_model)` internally.

**This is the fundamental mismatch**: `BulkStructuredModelEvaluator` is designed to take raw model instances and run comparisons. But in our use case, the per-document comparisons have **already happened** in the Evaluation Lambda. We just need to aggregate the pre-computed confusion matrices.

---

## Implementation: Direct Stickler Import

If we went with Option C (Docker image), here's what the implementation looks like:

### 1. Convert test_results_resolver to Docker image

```yaml
# nested/appsync/template.yaml
TestResultsResolverFunction:
  Type: AWS::Serverless::Function
  DependsOn: DockerBuildRun
  Metadata:
    SkipBuild: True
  Properties:
    PackageType: Image
    ImageUri: !Sub "${ECRRepository.RepositoryUri}:test-results-resolver-${ImageVersion}"
    Architectures:
      - arm64
    ImageConfig:
      Command:
        - "index.handler"
    MemorySize: 512
    Timeout: 300
```

New `requirements.txt`:
```
../../lib/idp_common_pkg[evaluation]
```

### 2. Use BulkStructuredModelEvaluator... but we can't

The `BulkStructuredModelEvaluator.update()` signature is:

```python
def update(self, gt_model: StructuredModel, pred_model: StructuredModel, doc_id: str = None):
```

It takes **StructuredModel instances**, not pre-computed confusion matrices. To use it, we'd need to:

1. Load the ground truth JSON from S3
2. Load the prediction JSON from S3
3. Reconstruct the Stickler schema (from the config)
4. Create `StructuredModel` instances from both
5. Call `update()` which internally calls `compare_with()` **again**

This means **re-running all per-document evaluations** in the test_results_resolver Lambda — duplicating work already done by the Evaluation Lambda.

### 3. Alternative: Use only the accumulation internals

We could import Stickler but only use its internal accumulation methods:

```python
from stickler.structured_object_evaluator.bulk_structured_model_evaluator import BulkStructuredModelEvaluator
from stickler.utils.process_evaluation import ProcessEvaluation

# Can't use update() — it needs StructuredModel instances
# Would need to call internal methods directly:
evaluator = BulkStructuredModelEvaluator.__new__(BulkStructuredModelEvaluator)
evaluator.reset()

for doc_result in eval_results:
    cm = doc_result.get("confusion_matrix", {})
    evaluator._accumulate_confusion_matrix(cm)  # ← Private method
    evaluator._processed_count += 1

result = evaluator.compute()
```

This works but:
- Depends on **private methods** (`_accumulate_confusion_matrix`, `_processed_count`)
- Could break on any Stickler version update
- Requires the full 221 MB dependency chain just to call a few accumulation functions

---

## Side-by-Side Comparison

| Aspect | Current Plan: Custom Aggregator | KISS Alt: Direct Stickler Import |
|--------|--------------------------------|----------------------------------|
| **New code** | ~80 lines (`aggregator.py`) | ~20 lines (import + wire up) |
| **Dependency size** | 0 MB (pure Python, no deps) | ~221 MB (stickler + scipy + pandas + numpy) |
| **Lambda packaging change** | None | Convert to Docker image OR create new layer |
| **Build pipeline change** | None | Add Docker build step for test_results_resolver |
| **CloudFormation changes** | Add `fieldLevelMetrics` to schema | Add `fieldLevelMetrics` + change Lambda to Image type |
| **Cold start impact** | None | +2-5s (loading scipy/numpy) |
| **Memory impact** | Negligible | +100-200 MB runtime |
| **API compatibility** | Operates on pre-computed confusion matrix dicts | `update()` needs StructuredModel instances (mismatch) |
| **Version coupling** | None — standalone code | Tied to Stickler version; private API risk |
| **Re-evaluation** | No — uses pre-computed results | Yes — would re-run `compare_with()` per doc |
| **Maintenance** | Own code to maintain (~80 lines) | Stickler maintains the logic |
| **Test complexity** | Unit test with dict fixtures | Need Stickler + StructuredModel in test env |
| **Deploy time impact** | None | +30-60s (Docker build + push) |

---

## Recommendation

**Stick with the custom `BulkEvaluationAggregator` approach.** Here's why:

### 1. Fundamental API mismatch
`BulkStructuredModelEvaluator.update()` takes `StructuredModel` instances and runs `compare_with()` internally. Our use case has **already-computed** confusion matrices. Using Stickler directly would mean either:
- Re-running all evaluations (wasteful, slow)
- Calling private methods (fragile)

### 2. Massive dependency overhead for minimal logic
The accumulation logic is ~80 lines of straightforward Python (sum counts, derive P/R/F1). Importing Stickler brings in **221 MB** of dependencies (scipy, pandas, numpy) that are never used by the accumulation code.

### 3. No packaging changes needed
The test_results_resolver is currently a simple Zip Lambda with no dependencies. Keeping it that way avoids:
- Docker image build pipeline changes
- New Lambda layer creation
- Increased cold start times
- Higher memory consumption

### 4. Stickler is still the source of truth
The per-document evaluation still uses Stickler's `compare_with()` in the Evaluation Lambda. The custom aggregator just sums up the confusion matrices that Stickler produced. If Stickler changes its confusion matrix format, we update the aggregator — same as we'd update any consumer.

### 5. The accumulation logic is stable
Summing TP/FP/FN/TN counts and deriving P/R/F1 is textbook ML metrics. This logic won't change. The risk of "getting it wrong" is minimal compared to the operational complexity of adding Stickler to the resolver Lambda.

---

## When Direct Import WOULD Make Sense

The Stickler import approach would be better if:

- We needed to **re-evaluate** documents with different thresholds/comparators at query time
- We needed Stickler's `merge_state()` for distributed evaluation across multiple Lambdas
- The test_results_resolver was **already** a Docker image with heavy dependencies
- Stickler offered a `update_from_confusion_matrix(cm_dict)` public method (it doesn't today)

If Stickler adds a public API for accumulating pre-computed confusion matrices in a future version, we should revisit this decision.

---

## Future: Post-Refactor Migration Path

When Stickler ships `aggregate_from_comparisons(comparisons: list[dict])`, the migration is:

1. **`idp_common` aggregator** — replace custom `BulkEvaluationAggregator` with a thin wrapper around Stickler's method. The `EvaluationFunction` Lambda (Docker image) already has Stickler installed.

2. **`test_results_resolver` Lambda** — keep the standalone `bulk_aggregator.py` copy. This Lambda is a bare Zip with no layers. Adding Stickler (221 MB) is not viable unless the dependency chain shrinks significantly.

3. **Long-term** — if Stickler extracts the accumulation logic into a zero-dep subpackage (e.g., `stickler-eval[core]` without scipy/pandas), the Lambda copy can be replaced with a direct import.

See [future-stickler-refactor.md](../future-stickler-refactor.md) for full details.
