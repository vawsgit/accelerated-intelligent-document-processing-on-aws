# GraphQL Schema & API Changes

## Current `TestRun` Type

```graphql
type TestRun @aws_cognito_user_pools @aws_iam {
  testRunId: String!
  testSetId: String
  testSetName: String
  status: String!
  filesCount: Int!
  completedFiles: Int
  failedFiles: Int
  overallAccuracy: Float
  weightedOverallScores: AWSJSON
  averageConfidence: Float
  accuracyBreakdown: AWSJSON
  splitClassificationMetrics: AWSJSON
  totalCost: Float
  costBreakdown: AWSJSON
  createdAt: AWSDateTime
  completedAt: AWSDateTime
  context: String
  config: AWSJSON
}
```

## Proposed Addition

```graphql
type TestRun @aws_cognito_user_pools @aws_iam {
  # ... all existing fields unchanged ...
  fieldLevelMetrics: AWSJSON    # ← NEW
}
```

## `fieldLevelMetrics` JSON Shape

The `AWSJSON` blob returned by `fieldLevelMetrics` has this structure:

```json
{
  "document_count": 75,
  "overall": {
    "tp": 450, "fp": 12, "fn": 8, "tn": 5,
    "precision": 0.974,
    "recall": 0.982,
    "f1": 0.978,
    "accuracy": 0.957
  },
  "fields": {
    "invoice_id": {
      "tp": 75, "fp": 0, "fn": 0, "tn": 0,
      "precision": 1.0, "recall": 1.0, "f1": 1.0, "accuracy": 1.0
    },
    "customer_name": {
      "tp": 68, "fp": 3, "fn": 4, "tn": 0,
      "precision": 0.957, "recall": 0.944, "f1": 0.951, "accuracy": 0.907
    },
    "line_items.description": {
      "tp": 180, "fp": 5, "fn": 2, "tn": 0,
      "precision": 0.973, "recall": 0.989, "f1": 0.981, "accuracy": 0.963
    }
  },
  "errors": []
}
```

## GraphQL Query Update

File: `src/ui/src/graphql/queries/getTestResults.js`

```javascript
// Current
const GET_TEST_RUN = `
  query GetTestRun($testRunId: String!) {
    getTestRun(testRunId: $testRunId) {
      testRunId
      testSetId
      testSetName
      status
      filesCount
      completedFiles
      failedFiles
      overallAccuracy
      weightedOverallScores
      averageConfidence
      accuracyBreakdown
      splitClassificationMetrics
      totalCost
      costBreakdown
      createdAt
      completedAt
      context
      config
    }
  }
`;

// Proposed — add fieldLevelMetrics
const GET_TEST_RUN = `
  query GetTestRun($testRunId: String!) {
    getTestRun(testRunId: $testRunId) {
      testRunId
      testSetId
      testSetName
      status
      filesCount
      completedFiles
      failedFiles
      overallAccuracy
      weightedOverallScores
      averageConfidence
      accuracyBreakdown
      splitClassificationMetrics
      totalCost
      costBreakdown
      createdAt
      completedAt
      context
      config
      fieldLevelMetrics          # ← NEW
    }
  }
`;
```

## Resolver Changes

File: `nested/appsync/src/lambda/test_results_resolver/index.py`

### Current `_aggregate_test_run_metrics`

```python
def _aggregate_test_run_metrics(test_run_id):
    evaluation_metrics = _get_evaluation_metrics_from_athena(test_run_id)
    cost_data = _get_cost_data_from_athena(test_run_id)
    return {
        'overall_accuracy': evaluation_metrics.get('overall_accuracy'),
        'weighted_overall_scores': evaluation_metrics.get('weighted_overall_scores', {}),
        'average_confidence': evaluation_metrics.get('average_confidence'),
        'accuracy_breakdown': evaluation_metrics.get('accuracy_breakdown', {}),
        'split_classification_metrics': evaluation_metrics.get('split_classification_metrics', {}),
        'total_cost': cost_data.get('total_cost', 0),
        'cost_breakdown': cost_data.get('cost_breakdown', {})
    }
```

### Proposed `_aggregate_test_run_metrics`

```python
def _aggregate_test_run_metrics(test_run_id):
    evaluation_metrics = _get_evaluation_metrics_from_athena(test_run_id)
    cost_data = _get_cost_data_from_athena(test_run_id)
    field_level_metrics = _get_field_level_metrics(test_run_id)  # ← NEW
    return {
        'overall_accuracy': evaluation_metrics.get('overall_accuracy'),
        'weighted_overall_scores': evaluation_metrics.get('weighted_overall_scores', {}),
        'average_confidence': evaluation_metrics.get('average_confidence'),
        'accuracy_breakdown': evaluation_metrics.get('accuracy_breakdown', {}),
        'split_classification_metrics': evaluation_metrics.get('split_classification_metrics', {}),
        'total_cost': cost_data.get('total_cost', 0),
        'cost_breakdown': cost_data.get('cost_breakdown', {}),
        'field_level_metrics': field_level_metrics,                # ← NEW
    }
```

### New function: `_get_field_level_metrics`

```python
def _get_field_level_metrics(test_run_id):
    """
    Load per-document eval JSONs from S3, extract confusion matrices,
    and aggregate via BulkEvaluationAggregator.
    """
    # 1. Query Athena for document IDs + eval result S3 URIs
    # 2. Read each eval JSON from S3
    # 3. Feed confusion_matrix into aggregator
    # 4. Return aggregator.compute()
```

### Cache update in `handle_cache_update_request`

```python
# Current cached fields
metrics_to_cache = {
    'overallAccuracy': ...,
    'weightedOverallScores': ...,
    'averageConfidence': ...,
    'accuracyBreakdown': ...,
    'splitClassificationMetrics': ...,
    'totalCost': ...,
    'costBreakdown': ...,
}

# Proposed — add fieldLevelMetrics
metrics_to_cache = {
    # ... all existing ...
    'fieldLevelMetrics': json.dumps(aggregated_metrics.get('field_level_metrics', {})),
}
```

## Backward Compatibility

- `fieldLevelMetrics` is nullable — returns `null` for test runs that don't have confusion matrix data
- All existing fields and queries remain unchanged
- UI gracefully handles missing data (conditional rendering)
