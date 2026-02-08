# UI Changes: Test Studio Field-Level Metrics

## Current Test Studio Layout

The `TestResults.jsx` component (1307 lines) currently renders:

```
┌─────────────────────────────────────────────┐
│ Test Run: <id>                    [Re-Run]  │
├─────────────────────────────────────────────┤
│ Summary: Overall Accuracy, Confidence, Cost │
├─────────────────────────────────────────────┤
│ ▸ Average Accuracy and Split Metrics        │
│   ├─ Weighted Overall Score                 │
│   ├─ Page/Split Accuracy                    │
│   └─ ▸ Additional Metrics (P/R/F1/FAR/FDR) │
├─────────────────────────────────────────────┤
│ ▸ Cost Breakdown                            │
│   └─ Per-service cost table                 │
├─────────────────────────────────────────────┤
│ Per-Document Scores (Bar/Line Chart)        │
│   └─ Clickable bars → document detail modal │
└─────────────────────────────────────────────┘
```

## Proposed Addition

Insert a new section **between** the accuracy breakdown and cost breakdown:

```
┌─────────────────────────────────────────────┐
│ ▸ Average Accuracy and Split Metrics        │
├─────────────────────────────────────────────┤
│ ▸ Field-Level Metrics (Aggregated)    ← NEW │
│   ├─ Overall: P=0.974 R=0.982 F1=0.978     │
│   ├─ Sortable table (worst F1 first)        │
│   └─ F1 bar chart per field                 │
├─────────────────────────────────────────────┤
│ ▸ Cost Breakdown                            │
└─────────────────────────────────────────────┘
```

## Field-Level Metrics Table

Sortable Cloudscape `Table` component, default sorted by F1 ascending (worst first):

| Field | F1 | Precision | Recall | TP | FP | FN |
|-------|-----|-----------|--------|-----|-----|-----|
| line_items.amount | 🔴 0.667 | 0.667 | 0.667 | 2 | 1 | 1 |
| customer_name | 🟡 0.800 | 0.667 | 1.000 | 2 | 1 | 0 |
| invoice_id | 🟢 1.000 | 1.000 | 1.000 | 3 | 0 | 0 |
| total_amount | 🟢 1.000 | 1.000 | 1.000 | 3 | 0 | 0 |

### Color Coding

| F1 Range | Color | Badge |
|----------|-------|-------|
| < 0.5 | Red | 🔴 `<Badge color="red">` |
| 0.5 – 0.8 | Yellow | 🟡 `<Badge color="blue">` (Cloudscape doesn't have yellow) |
| > 0.8 | Green | 🟢 `<Badge color="green">` |

## F1 Bar Chart

Horizontal bar chart using existing `recharts` dependency (already imported in TestResults.jsx):

```
invoice_id        ████████████████████████████████ 1.000
total_amount      ████████████████████████████████ 1.000
customer_name     █████████████████████████▒▒▒▒▒▒ 0.800
line_items.amount ████████████████████▒▒▒▒▒▒▒▒▒▒▒ 0.667
```

- Sorted by F1 descending for the chart (visual: best at top)
- Color-coded bars matching the table badge colors
- Tooltip shows full metrics on hover

## Component Structure

```jsx
// New component in TestResults.jsx
const FieldLevelMetrics = ({ fieldLevelMetrics }) => {
  // Parse AWSJSON string
  // Render ExpandableSection with:
  //   1. Overall summary (ColumnLayout with P/R/F1 boxes)
  //   2. Sortable Table (field rows)
  //   3. ExpandableSection with BarChart
};

// Integration point in TestResults render:
<SpaceBetween direction="vertical" size="l">
  <ComprehensiveBreakdown ... />
  <FieldLevelMetrics fieldLevelMetrics={results.fieldLevelMetrics} />  {/* ← NEW */}
  {/* Cost breakdown */}
  {/* Per-document chart */}
</SpaceBetween>
```

## Data Parsing

The `fieldLevelMetrics` comes as an `AWSJSON` string from AppSync. It needs `JSON.parse()`:

```javascript
// In the useEffect that fetches test run data:
const parsedResults = {
  ...data.getTestRun,
  fieldLevelMetrics: data.getTestRun.fieldLevelMetrics
    ? JSON.parse(data.getTestRun.fieldLevelMetrics)
    : null,
};
```

## Conditional Rendering

The section only renders when `fieldLevelMetrics` is non-null and has fields:

```jsx
{fieldLevelMetrics?.fields && Object.keys(fieldLevelMetrics.fields).length > 0 && (
  <FieldLevelMetrics fieldLevelMetrics={fieldLevelMetrics} />
)}
```

This ensures backward compatibility — older test runs without confusion matrix data simply don't show the section.

## Cloudscape Components Used

All components are already imported in `TestResults.jsx`:

| Component | Usage |
|-----------|-------|
| `ExpandableSection` | Collapsible container for the section |
| `Table` | Sortable field metrics table |
| `Header` | Section header |
| `ColumnLayout` | Overall metrics summary boxes |
| `Box` | Metric value display |
| `Badge` | Color-coded F1 indicators |
| `BarChart` (recharts) | F1 per-field visualization |
