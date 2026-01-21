# LLM Council: Quality Assurance Guide

## Overview

This document describes a simple, maintainable QA strategy for monitoring the LLM Council's performance over time. The approach prioritizes:

- **Simplicity**: Minimal code, easy to understand
- **Interpretability**: Clear metrics anyone can review
- **Low maintenance**: Mostly automated with periodic manual checks
- **Early detection**: Catch issues before they become problems

---

## QA Pyramid

```
                    ▲
                   /|\
                  / | \
                 /  |  \      MONTHLY
                / Spot  \     Manual review of 5 samples
               /  Check  \
              /───────────\
             /             \   WEEKLY
            /   Dashboard   \  Review automated metrics
           /                 \
          /───────────────────\
         /                     \  CONTINUOUS
        /   Automated Logging   \ Every query logged
       /                         \
      /───────────────────────────\
     /                             \  ON CHANGE
    /     Regression Tests          \ Run on code changes
   /─────────────────────────────────\
```

---

## 1. Automated Logging (Continuous)

### Session Log Schema

Every council query should log:

```typescript
interface SessionLog {
  // Identification
  id: string;
  timestamp: string;
  conversation_id: string;

  // Query info
  query: string;
  query_length: number;
  query_type?: 'technical' | 'opinion' | 'factual' | 'creative';

  // Stage 1 metrics
  stage1: {
    models_queried: number;
    models_succeeded: number;
    failures: string[];           // Model names that failed
    avg_response_length: number;
    duration_ms: number;
  };

  // Stage 2 metrics
  stage2: {
    models_queried: number;
    models_succeeded: number;
    parse_successes: number;      // How many rankings parsed correctly
    ranking_variance: number;     // Disagreement measure
    self_rank_positions: Record<string, number>;  // Where each model ranked itself
    duration_ms: number;
  };

  // Stage 3 metrics
  stage3: {
    chairman_model: string;
    response_length: number;
    duration_ms: number;
  };

  // Aggregate rankings
  aggregate: {
    winner: string;
    winner_avg_rank: number;
    spread: number;               // Difference between best and worst
  };

  // Totals
  total_duration_ms: number;

  // Optional: user feedback (added later)
  feedback?: {
    rating?: 1 | 2 | 3 | 4 | 5;
    helpful?: boolean;
    viewed_stage1?: boolean;
    viewed_stage2?: boolean;
    comment?: string;
  };
}
```

### Storage Format

Use JSON Lines (`.jsonl`) for easy appending and processing:

```
data/
├── conversations/          # Existing conversation storage
└── analytics/
    ├── sessions.jsonl      # One JSON object per line
    └── daily/
        ├── 2026-01-20.jsonl
        ├── 2026-01-21.jsonl
        └── ...
```

### Implementation

```typescript
// services/analytics.ts
import { appendFile, mkdir } from 'fs/promises';
import { existsSync } from 'fs';

const ANALYTICS_DIR = 'data/analytics';

export async function logSession(log: SessionLog): Promise<void> {
  await ensureDir(ANALYTICS_DIR);

  const date = new Date().toISOString().split('T')[0];
  const dailyFile = `${ANALYTICS_DIR}/daily/${date}.jsonl`;
  const allFile = `${ANALYTICS_DIR}/sessions.jsonl`;

  const line = JSON.stringify(log) + '\n';

  await Promise.all([
    appendFile(dailyFile, line),
    appendFile(allFile, line),
  ]);
}

async function ensureDir(dir: string): Promise<void> {
  if (!existsSync(dir)) {
    await mkdir(dir, { recursive: true });
  }
  if (!existsSync(`${dir}/daily`)) {
    await mkdir(`${dir}/daily`, { recursive: true });
  }
}
```

### Key Metrics to Calculate

```typescript
// Calculate ranking variance (measure of disagreement)
export function calculateRankingVariance(
  stage2Results: Stage2Result[],
  labelToModel: Record<string, string>
): number {
  const modelPositions: Record<string, number[]> = {};

  // Collect all positions for each model
  for (const result of stage2Results) {
    result.parsed_ranking.forEach((label, position) => {
      const model = labelToModel[label];
      if (model) {
        modelPositions[model] = modelPositions[model] || [];
        modelPositions[model].push(position + 1);
      }
    });
  }

  // Calculate variance for each model, then average
  let totalVariance = 0;
  let modelCount = 0;

  for (const positions of Object.values(modelPositions)) {
    if (positions.length > 1) {
      const mean = positions.reduce((a, b) => a + b, 0) / positions.length;
      const variance = positions.reduce((sum, p) => sum + (p - mean) ** 2, 0) / positions.length;
      totalVariance += variance;
      modelCount++;
    }
  }

  return modelCount > 0 ? totalVariance / modelCount : 0;
}

// Calculate where models ranked themselves (self-promotion detection)
export function calculateSelfRankPositions(
  stage2Results: Stage2Result[],
  labelToModel: Record<string, string>
): Record<string, number> {
  const selfRanks: Record<string, number> = {};

  // Invert the mapping
  const modelToLabel: Record<string, string> = {};
  for (const [label, model] of Object.entries(labelToModel)) {
    modelToLabel[model] = label;
  }

  for (const result of stage2Results) {
    const evaluatorModel = result.model;
    const evaluatorLabel = modelToLabel[evaluatorModel];

    const selfPosition = result.parsed_ranking.indexOf(evaluatorLabel);
    if (selfPosition !== -1) {
      selfRanks[evaluatorModel] = selfPosition + 1; // 1-indexed
    }
  }

  return selfRanks;
}
```

---

## 2. Health Indicators

### Thresholds

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| **Model success rate** | >95% | 80-95% | <80% |
| **Ranking parse rate** | >95% | 85-95% | <85% |
| **Ranking variance** | 0.5-2.5 | 0.2-0.5 or 2.5-3.5 | <0.2 or >3.5 |
| **Self-rank avg** | 2.0-3.0 | 1.5-2.0 or 3.0-3.5 | <1.5 or >3.5 |
| **Response time** | <90s | 90-180s | >180s |
| **Chairman success** | 100% | 95-100% | <95% |

### Interpretation Guide

**Ranking variance:**
- **Too low (<0.2)**: Suspiciously uniform — models may be copying patterns
- **Healthy (0.5-2.5)**: Normal disagreement on subjective questions
- **Too high (>3.5)**: No consensus — question may be unanswerable or poorly formed

**Self-rank average:**
- **Too low (<1.5)**: Models consistently rank themselves #1 — self-promotion bias
- **Healthy (2.0-3.0)**: Models evaluate themselves fairly
- **Too high (>3.5)**: Models may be overly self-critical or response quality issues

**Model success rate drops:**
- Check OpenRouter status page
- Check API key validity/quota
- Check for model deprecation

---

## 3. Weekly Dashboard

### Report Template

```markdown
# LLM Council Health Report
Week of: {START_DATE} to {END_DATE}
Generated: {TIMESTAMP}

## Summary
- Total queries: {TOTAL}
- Avg response time: {AVG_TIME}s
- Overall health: {HEALTHY|WARNING|CRITICAL}

## Model Performance

| Model | Success | Avg Rank | Self-Rank | Status |
|-------|---------|----------|-----------|--------|
| GPT-5.1 | {X}% | {X.X} | {X.X} | {✓|⚠|✗} |
| Gemini | {X}% | {X.X} | {X.X} | {✓|⚠|✗} |
| Claude | {X}% | {X.X} | {X.X} | {✓|⚠|✗} |
| Grok | {X}% | {X.X} | {X.X} | {✓|⚠|✗} |

## Consensus Quality

| Category | Count | Percentage |
|----------|-------|------------|
| Strong (variance <1.0) | {X} | {X}% |
| Moderate (1.0-2.0) | {X} | {X}% |
| Weak (>2.0) | {X} | {X}% |

## Parsing Health
- Ranking parse success: {X}%
- Failed parses this week: {X}

## Timing Breakdown
- Stage 1 avg: {X}s
- Stage 2 avg: {X}s
- Stage 3 avg: {X}s
- Total avg: {X}s

## Alerts
{LIST_OF_ALERTS}

## Trends (vs last week)
- Query volume: {↑|↓|→} {X}%
- Response time: {↑|↓|→} {X}%
- Success rate: {↑|↓|→} {X}%
```

### Report Generation Script

```bash
#!/bin/bash
# scripts/generate-weekly-report.sh

set -e

ANALYTICS_DIR="data/analytics"
REPORT_DIR="data/reports"
mkdir -p "$REPORT_DIR"

# Date range
END_DATE=$(date +%Y-%m-%d)
START_DATE=$(date -d "7 days ago" +%Y-%m-%d)
REPORT_FILE="$REPORT_DIR/weekly-${END_DATE}.md"

echo "Generating report for $START_DATE to $END_DATE..."

# Aggregate last 7 days of logs
cat $ANALYTICS_DIR/daily/{$START_DATE..$END_DATE}.jsonl 2>/dev/null | \
  jq -s '
    {
      total: length,
      avg_duration: (map(.total_duration_ms) | add / length / 1000),
      model_success: (
        group_by(.stage1.failures[]) |
        map({key: .[0].stage1.failures[0], value: length})
      ),
      parse_success_rate: (
        map(.stage2.parse_successes) | add
      ) / (
        map(.stage2.models_succeeded) | add
      ) * 100,
      avg_variance: (map(.stage2.ranking_variance) | add / length)
    }
  ' > /tmp/metrics.json

# Generate markdown report
cat << EOF > "$REPORT_FILE"
# LLM Council Health Report
Week of: $START_DATE to $END_DATE
Generated: $(date -Iseconds)

## Summary
- Total queries: $(jq .total /tmp/metrics.json)
- Avg response time: $(jq '.avg_duration | round' /tmp/metrics.json)s
- Parse success rate: $(jq '.parse_success_rate | round' /tmp/metrics.json)%
- Avg ranking variance: $(jq '.avg_variance | . * 100 | round / 100' /tmp/metrics.json)

EOF

echo "Report saved to $REPORT_FILE"
```

---

## 4. Monthly Spot Checks

### Purpose

Automated metrics can miss qualitative issues. Monthly manual review catches:
- Factually incorrect answers that models agreed on
- Missed perspectives in synthesis
- Subtle bias patterns
- Prompt degradation over time

### Spot Check Protocol

**Sample selection:** Randomly select 5 queries from the past month.

**For each query, evaluate:**

```markdown
## Spot Check #{N}

**Query ID:** {ID}
**Date:** {DATE}
**Query:** "{QUERY_TEXT}"

### Stage 1 Evaluation
- [ ] All models addressed the core question
- [ ] Responses were substantively different (not paraphrasing each other)
- [ ] No obvious factual errors detected
- [ ] Response lengths were appropriate

Issues found: _______________

### Stage 2 Evaluation
- [ ] Rankings seem reasonable given response quality
- [ ] Evaluations cite specific evidence from responses
- [ ] No obvious brand/model bias detected
- [ ] Parsing extracted correct ranking order

Issues found: _______________

### Stage 3 Evaluation
- [ ] Synthesis incorporates insights from multiple responses
- [ ] Highly-ranked responses influenced the final answer
- [ ] Criticisms from evaluations were addressed
- [ ] Final answer is actionable and complete

Issues found: _______________

### Overall Assessment
- [ ] PASS - No significant issues
- [ ] MINOR ISSUES - Noted but acceptable
- [ ] MAJOR ISSUES - Requires investigation
- [ ] FAIL - System malfunction

**Notes:**
_______________

**Recommendations:**
_______________
```

### Tracking Spot Check Results

```
data/
└── qa/
    └── spot-checks/
        ├── 2026-01.md
        ├── 2026-02.md
        └── ...
```

Monthly summary:
```markdown
# Spot Check Summary - January 2026

Checks performed: 5
Pass: 4
Minor issues: 1
Major issues: 0
Fail: 0

## Issues Found
1. Query #abc123: Stage 2 evaluations were too brief (3 of 4 models)
   - Recommendation: Consider adjusting prompt to request more detailed reasoning

## Trends
- Synthesis quality: Stable
- Ranking accuracy: Stable
- Parse reliability: Improved (was having issues in December)
```

---

## 5. Regression Tests

### Purpose

Catch issues when code changes. Run automatically on:
- Pull requests
- Pre-deployment
- Weekly scheduled runs

### Test Suite

```typescript
// tests/regression.test.ts
import { describe, test, expect } from 'bun:test';
import { runFullCouncil } from '../packages/backend/src/services/council';

describe('LLM Council Regression Tests', () => {

  test('trivial factual query - all models should agree', async () => {
    const result = await runFullCouncil('What is 2 + 2?');

    // All models should respond
    expect(result.stage1.length).toBe(4);

    // All responses should mention "4"
    for (const response of result.stage1) {
      expect(response.response.toLowerCase()).toContain('4');
    }

    // Low variance expected (everyone agrees)
    expect(result.metadata.aggregate_rankings[0].average_rank).toBeLessThan(2);
  }, 120000); // 2 min timeout

  test('subjective query - should have ranking variance', async () => {
    const result = await runFullCouncil('Is Python or JavaScript better for web development?');

    // Should have some disagreement
    const variance = calculateVariance(result.stage2);
    expect(variance).toBeGreaterThan(0.3);

    // Synthesis should acknowledge trade-offs
    const synthesis = result.stage3.response.toLowerCase();
    expect(
      synthesis.includes('depends') ||
      synthesis.includes('trade-off') ||
      synthesis.includes('both')
    ).toBe(true);
  }, 120000);

  test('all rankings should parse successfully', async () => {
    const result = await runFullCouncil('Explain the benefits of exercise');

    // All Stage 2 results should have parsed rankings
    for (const ranking of result.stage2) {
      expect(ranking.parsed_ranking.length).toBeGreaterThan(0);
      expect(ranking.parsed_ranking.length).toBeLessThanOrEqual(4);
    }
  }, 120000);

  test('response times within acceptable range', async () => {
    const start = Date.now();
    await runFullCouncil('What is machine learning?');
    const duration = Date.now() - start;

    // Should complete within 3 minutes
    expect(duration).toBeLessThan(180000);
  }, 200000);

  test('handles model failure gracefully', async () => {
    // This would require mocking a model failure
    // Simplified: just ensure the system doesn't crash with partial results
    const result = await runFullCouncil('Test query');

    // Should have at least some results even if one model fails
    expect(result.stage1.length).toBeGreaterThanOrEqual(1);
    expect(result.stage3.response.length).toBeGreaterThan(0);
  }, 120000);

});
```

### Running Tests

```bash
# Run all regression tests
bun test tests/regression.test.ts

# Run with verbose output
bun test tests/regression.test.ts --verbose

# Run specific test
bun test tests/regression.test.ts -t "trivial factual"
```

### CI Integration

```yaml
# .github/workflows/qa.yml
name: QA Tests

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 0 * * 0'  # Weekly on Sunday

jobs:
  regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1

      - name: Install dependencies
        run: bun install

      - name: Run regression tests
        env:
          OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
        run: bun test tests/regression.test.ts

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results/
```

---

## 6. User Feedback (Optional)

### Simple Feedback UI

Add to Stage 3 component:

```tsx
// components/Feedback.tsx
import { useState } from 'preact/hooks';

interface Props {
  sessionId: string;
}

export function Feedback({ sessionId }: Props) {
  const [submitted, setSubmitted] = useState(false);

  const submit = async (helpful: boolean) => {
    await fetch('/api/feedback', {
      method: 'POST',
      body: JSON.stringify({ sessionId, helpful }),
    });
    setSubmitted(true);
  };

  if (submitted) {
    return <div class="feedback-thanks">Thanks for your feedback!</div>;
  }

  return (
    <div class="feedback">
      <span>Was this helpful?</span>
      <button onClick={() => submit(true)}>👍 Yes</button>
      <button onClick={() => submit(false)}>👎 No</button>
    </div>
  );
}
```

### Feedback Metrics

Track weekly:
- **Helpful rate**: `thumbs_up / (thumbs_up + thumbs_down)`
- **Feedback rate**: `total_feedback / total_queries`
- **Negative feedback patterns**: Common issues in "No" responses

---

## 7. Alerting

### Simple Alert Rules

```typescript
// services/alerts.ts
interface Alert {
  level: 'warning' | 'critical';
  metric: string;
  value: number;
  threshold: number;
  message: string;
}

export function checkHealth(metrics: DailyMetrics): Alert[] {
  const alerts: Alert[] = [];

  // Model failure rate
  if (metrics.modelFailureRate > 0.2) {
    alerts.push({
      level: 'critical',
      metric: 'model_failure_rate',
      value: metrics.modelFailureRate,
      threshold: 0.2,
      message: `Model failure rate ${(metrics.modelFailureRate * 100).toFixed(1)}% exceeds 20%`,
    });
  } else if (metrics.modelFailureRate > 0.05) {
    alerts.push({
      level: 'warning',
      metric: 'model_failure_rate',
      value: metrics.modelFailureRate,
      threshold: 0.05,
      message: `Model failure rate ${(metrics.modelFailureRate * 100).toFixed(1)}% exceeds 5%`,
    });
  }

  // Parse success rate
  if (metrics.parseSuccessRate < 0.85) {
    alerts.push({
      level: 'critical',
      metric: 'parse_success_rate',
      value: metrics.parseSuccessRate,
      threshold: 0.85,
      message: `Ranking parse rate ${(metrics.parseSuccessRate * 100).toFixed(1)}% below 85%`,
    });
  }

  // Response time
  if (metrics.avgResponseTime > 180) {
    alerts.push({
      level: 'warning',
      metric: 'avg_response_time',
      value: metrics.avgResponseTime,
      threshold: 180,
      message: `Avg response time ${metrics.avgResponseTime.toFixed(0)}s exceeds 3 minutes`,
    });
  }

  // Self-ranking bias
  for (const [model, avgSelfRank] of Object.entries(metrics.avgSelfRank)) {
    if (avgSelfRank < 1.5) {
      alerts.push({
        level: 'warning',
        metric: 'self_rank_bias',
        value: avgSelfRank,
        threshold: 1.5,
        message: `${model} self-ranking avg ${avgSelfRank.toFixed(2)} indicates possible self-promotion`,
      });
    }
  }

  return alerts;
}
```

### Notification Options

- **Email**: Weekly digest with alerts
- **Slack/Discord**: Real-time critical alerts
- **Log file**: All alerts to `data/alerts.log`

---

## Implementation Checklist

### Phase 1: Logging (2 hours)
- [ ] Create `services/analytics.ts`
- [ ] Add logging calls to council flow
- [ ] Create `data/analytics/` directory structure
- [ ] Test logging works

### Phase 2: Dashboard (4 hours)
- [ ] Create report generation script
- [ ] Define health thresholds
- [ ] Generate first weekly report
- [ ] Set up weekly cron job

### Phase 3: Regression Tests (2 hours)
- [ ] Create test file with 5 basic tests
- [ ] Run tests locally
- [ ] Add to CI pipeline (optional)

### Phase 4: Spot Check Protocol (1 hour)
- [ ] Create spot check template
- [ ] Document in this file
- [ ] Schedule monthly calendar reminder

### Phase 5: Alerts (2 hours, optional)
- [ ] Implement alert checking
- [ ] Choose notification method
- [ ] Test alert triggering

### Phase 6: User Feedback (2 hours, optional)
- [ ] Add feedback UI component
- [ ] Create feedback API endpoint
- [ ] Add to analytics logging

---

## Maintenance Schedule

| Task | Frequency | Time Required |
|------|-----------|---------------|
| Review weekly dashboard | Weekly | 10 minutes |
| Spot check queries | Monthly | 1 hour |
| Review regression test results | On failure | 30 minutes |
| Update thresholds | Quarterly | 30 minutes |
| Full QA audit | Annually | 4 hours |
