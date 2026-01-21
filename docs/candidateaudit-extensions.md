# CandidateAudit.com Integration Extensions

This document outlines how LLM Council can be used as a "stealth alternative reality check" for candidateaudit.com's decision-making processes.

---

## Overview

### Purpose

Use LLM Council as an independent validation layer to audit decisions made by candidateaudit.com. The council provides:

1. **Ranking validation** — Do multiple LLMs agree with the candidate rankings?
2. **Reasoning validation** — Is the reasoning behind decisions sound?
3. **Disagreement flagging** — Highlight cases where the council disagrees with the primary decision

### Integration Models

Two integration approaches are available:

| Model | Description | Best For |
|-------|-------------|----------|
| **Batch Audit** | Export decisions, audit separately | QA, process refinement, low complexity |
| **Real-time API** | Validate during decision-making | Live validation, user-facing feedback |

---

## Integration Model Comparison

| Factor | Batch Audit | Real-time API |
|--------|-------------|---------------|
| **Complexity** | Lower — separate system | Higher — integrated |
| **Latency** | N/A (async) | +30-60s per decision |
| **User experience** | Review after the fact | Immediate feedback |
| **Cost** | Pay per audit run | Pay per decision |
| **candidateaudit.com changes** | Export only | API integration |
| **Use case** | QA, process refinement | Live validation |

---

# Option A: Batch Audit

Export decisions from candidateaudit.com and audit them separately:

```
candidateaudit.com → exports decisions → LLM Council batch audit → audit report
```

This keeps candidateaudit.com simple while providing sophisticated validation.

---

## Batch Audit Architecture

### System Flow

```
┌─────────────────────┐     Export      ┌──────────────────────┐
│  candidateaudit.com │ ───────────────►│    Decision Export   │
│                     │                 │    (JSON/CSV)        │
└─────────────────────┘                 └──────────┬───────────┘
                                                   │
                                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      LLM Council Batch Audit                     │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Stage 1:     │  │ Stage 2:     │  │ Stage 3:             │   │
│  │ Independent  │──│ Peer         │──│ Synthesis +          │   │
│  │ Evaluation   │  │ Ranking      │  │ Disagreement Check   │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
                                      ┌────────────────────────┐
                                      │     Audit Report       │
                                      │  - Agreement score     │
                                      │  - Flagged decisions   │
                                      │  - Reasoning analysis  │
                                      └────────────────────────┘
```

### Data Model

**Input: Decision Export from candidateaudit.com**

```typescript
interface CandidateDecision {
  id: string;
  timestamp: string;

  // The evaluation context
  job_description: string;
  evaluation_criteria: string[];

  // Candidates evaluated
  candidates: {
    id: string;
    name: string;
    resume_summary: string;
    interview_notes?: string;
  }[];

  // The decision made
  ranking: string[];           // Candidate IDs in order
  reasoning: string;           // Why this ranking was chosen
  confidence_score?: number;   // 0-1 if available
}
```

**Output: Audit Report**

```typescript
interface AuditReport {
  decision_id: string;
  audited_at: string;

  // Agreement metrics
  ranking_agreement: {
    council_ranking: string[];           // Council's consensus ranking
    original_ranking: string[];          // From candidateaudit.com
    kendall_tau: number;                 // Rank correlation (-1 to 1)
    exact_match: boolean;                // Perfect agreement?
    top_pick_match: boolean;             // #1 candidate matches?
  };

  // Reasoning analysis
  reasoning_analysis: {
    original_reasoning: string;
    council_reasoning: string;           // Stage 3 synthesis
    reasoning_alignment: 'aligned' | 'partial' | 'divergent';
    missing_considerations: string[];    // Things council considered that original didn't
    questionable_assumptions: string[];  // Potential issues council identified
  };

  // Overall assessment
  flag_status: 'confirmed' | 'review_recommended' | 'significant_disagreement';
  flag_reasons: string[];
  confidence_adjustment?: number;        // Suggested confidence change

  // Raw council data for transparency
  council_data: {
    stage1_responses: object[];
    stage2_rankings: object[];
    aggregate_rankings: object[];
    stage3_synthesis: string;
  };
}
```

---

## API Specification

### Batch Audit Endpoint

```typescript
POST /api/audit/batch
Content-Type: application/json
Authorization: Bearer <token>

{
  "decisions": CandidateDecision[],
  "options": {
    "model_tier": "premium" | "budget",
    "flag_threshold": number,        // Kendall tau below this triggers flag (default: 0.6)
    "include_raw_council": boolean,  // Include full council data in response
    "parallel_limit": number         // Max concurrent audits (default: 3)
  }
}
```

**Response (SSE Stream):**

```
event: audit_start
data: {"decision_id": "123", "index": 0, "total": 5}

event: audit_progress
data: {"decision_id": "123", "stage": "stage1_complete"}

event: audit_complete
data: {"decision_id": "123", "report": {...AuditReport}}

event: audit_start
data: {"decision_id": "456", "index": 1, "total": 5}

... continues for each decision ...

event: batch_complete
data: {"total": 5, "flagged": 2, "confirmed": 3}
```

### Single Decision Audit

```typescript
POST /api/audit/single
Content-Type: application/json
Authorization: Bearer <token>

{
  "decision": CandidateDecision,
  "options": { ... }
}
```

### Retrieve Audit History

```typescript
GET /api/audit/history?from=2026-01-01&to=2026-01-31&flagged_only=true
Authorization: Bearer <token>
```

---

## Council Prompt Engineering

### Stage 1: Independent Candidate Evaluation

Each council member evaluates candidates independently:

```typescript
const stage1Prompt = `
You are evaluating candidates for a position.

JOB DESCRIPTION:
${decision.job_description}

EVALUATION CRITERIA:
${decision.evaluation_criteria.map((c, i) => `${i + 1}. ${c}`).join('\n')}

CANDIDATES:
${decision.candidates.map(c => `
--- ${c.name} ---
${c.resume_summary}
${c.interview_notes ? `Interview Notes: ${c.interview_notes}` : ''}
`).join('\n')}

Please evaluate each candidate against the criteria, then provide your ranking from best to worst fit. Explain your reasoning for each candidate's placement.

End with:
FINAL RANKING:
1. [Candidate Name]
2. [Candidate Name]
...
`;
```

### Stage 2: Anonymized Peer Review

Standard council peer review, comparing Stage 1 evaluations.

### Stage 3: Synthesis with Comparison

The chairman synthesizes AND compares to the original decision:

```typescript
const stage3Prompt = `
Based on the council's deliberation, synthesize the final recommendation.

Additionally, compare the council's consensus to this original decision:

ORIGINAL RANKING:
${decision.ranking.map((id, i) => `${i + 1}. ${getCandidate(id).name}`).join('\n')}

ORIGINAL REASONING:
${decision.reasoning}

In your synthesis:
1. Present the council's consensus ranking and reasoning
2. Identify where the council agrees with the original decision
3. Identify where the council disagrees and why
4. Note any considerations the council raised that weren't in the original reasoning
5. Assess overall confidence in the original decision

End with a structured assessment:
AGREEMENT_LEVEL: [HIGH | MEDIUM | LOW]
KEY_AGREEMENTS: [bullet points]
KEY_DISAGREEMENTS: [bullet points]
MISSING_CONSIDERATIONS: [bullet points]
RECOMMENDATION: [CONFIRM | REVIEW | REJECT]
`;
```

---

## Disagreement Detection Logic

### Ranking Disagreement

```typescript
function calculateRankingAgreement(
  councilRanking: string[],
  originalRanking: string[]
): RankingAgreement {
  // Kendall Tau correlation coefficient
  const tau = kendallTauDistance(councilRanking, originalRanking);

  // Check top picks
  const topPickMatch = councilRanking[0] === originalRanking[0];
  const exactMatch = arraysEqual(councilRanking, originalRanking);

  return {
    council_ranking: councilRanking,
    original_ranking: originalRanking,
    kendall_tau: tau,
    exact_match: exactMatch,
    top_pick_match: topPickMatch
  };
}

function kendallTauDistance(a: string[], b: string[]): number {
  // Returns value from -1 (perfect disagreement) to 1 (perfect agreement)
  let concordant = 0;
  let discordant = 0;

  for (let i = 0; i < a.length; i++) {
    for (let j = i + 1; j < a.length; j++) {
      const aOrder = a.indexOf(a[i]) < a.indexOf(a[j]);
      const bOrder = b.indexOf(a[i]) < b.indexOf(a[j]);

      if (aOrder === bOrder) concordant++;
      else discordant++;
    }
  }

  const total = concordant + discordant;
  return total === 0 ? 1 : (concordant - discordant) / total;
}
```

### Reasoning Disagreement

Extracted from Stage 3 chairman synthesis:

```typescript
function analyzeReasoningAgreement(
  stage3Response: string
): ReasoningAnalysis {
  // Parse structured output from Stage 3
  const agreementLevel = extractField(stage3Response, 'AGREEMENT_LEVEL');
  const keyAgreements = extractList(stage3Response, 'KEY_AGREEMENTS');
  const keyDisagreements = extractList(stage3Response, 'KEY_DISAGREEMENTS');
  const missingConsiderations = extractList(stage3Response, 'MISSING_CONSIDERATIONS');
  const recommendation = extractField(stage3Response, 'RECOMMENDATION');

  return {
    reasoning_alignment: mapAgreementLevel(agreementLevel),
    missing_considerations: missingConsiderations,
    questionable_assumptions: keyDisagreements.filter(d =>
      d.toLowerCase().includes('assumption') ||
      d.toLowerCase().includes('questionable')
    ),
    council_reasoning: stage3Response
  };
}

function mapAgreementLevel(level: string): 'aligned' | 'partial' | 'divergent' {
  switch (level.toUpperCase()) {
    case 'HIGH': return 'aligned';
    case 'MEDIUM': return 'partial';
    case 'LOW': return 'divergent';
    default: return 'partial';
  }
}
```

### Flag Determination

```typescript
function determineFlagStatus(
  rankingAgreement: RankingAgreement,
  reasoningAnalysis: ReasoningAnalysis,
  options: AuditOptions
): { status: FlagStatus; reasons: string[] } {
  const reasons: string[] = [];

  // Ranking-based flags
  if (!rankingAgreement.top_pick_match) {
    reasons.push(`Top candidate disagreement: Council chose ${rankingAgreement.council_ranking[0]} vs original ${rankingAgreement.original_ranking[0]}`);
  }

  if (rankingAgreement.kendall_tau < options.flag_threshold) {
    reasons.push(`Low ranking correlation: ${rankingAgreement.kendall_tau.toFixed(2)} (threshold: ${options.flag_threshold})`);
  }

  // Reasoning-based flags
  if (reasoningAnalysis.reasoning_alignment === 'divergent') {
    reasons.push('Council reasoning significantly diverges from original');
  }

  if (reasoningAnalysis.missing_considerations.length > 2) {
    reasons.push(`${reasoningAnalysis.missing_considerations.length} considerations not addressed in original reasoning`);
  }

  if (reasoningAnalysis.questionable_assumptions.length > 0) {
    reasons.push(`Council identified ${reasoningAnalysis.questionable_assumptions.length} questionable assumptions`);
  }

  // Determine status
  let status: FlagStatus;
  if (reasons.length === 0) {
    status = 'confirmed';
  } else if (reasons.length <= 2 && rankingAgreement.top_pick_match) {
    status = 'review_recommended';
  } else {
    status = 'significant_disagreement';
  }

  return { status, reasons };
}
```

---

## Database Schema Extensions

```sql
-- Audit jobs
CREATE TABLE audit_jobs (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  status TEXT NOT NULL DEFAULT 'pending',  -- pending | processing | complete | failed
  total_decisions INTEGER NOT NULL,
  completed_decisions INTEGER NOT NULL DEFAULT 0,
  flagged_count INTEGER NOT NULL DEFAULT 0,
  options TEXT,  -- JSON
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  completed_at TEXT
);

CREATE INDEX idx_audit_jobs_user ON audit_jobs(user_id);
CREATE INDEX idx_audit_jobs_status ON audit_jobs(status);

-- Individual audit results
CREATE TABLE audit_results (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL REFERENCES audit_jobs(id) ON DELETE CASCADE,
  decision_id TEXT NOT NULL,  -- ID from candidateaudit.com

  -- Input
  decision_data TEXT NOT NULL,  -- JSON: CandidateDecision

  -- Output
  report TEXT,  -- JSON: AuditReport (null until complete)
  flag_status TEXT,

  -- Council data (if include_raw_council=true)
  council_data TEXT,  -- JSON

  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  completed_at TEXT
);

CREATE INDEX idx_audit_results_job ON audit_results(job_id);
CREATE INDEX idx_audit_results_flag ON audit_results(flag_status);
```

---

## Example Workflow

### 1. Export from candidateaudit.com

```json
{
  "decisions": [
    {
      "id": "hire-2026-001",
      "timestamp": "2026-01-20T14:30:00Z",
      "job_description": "Senior Software Engineer - Backend",
      "evaluation_criteria": [
        "5+ years backend experience",
        "Distributed systems knowledge",
        "Strong communication skills",
        "Leadership potential"
      ],
      "candidates": [
        {
          "id": "c1",
          "name": "Alice Chen",
          "resume_summary": "8 years at FAANG, led team of 12..."
        },
        {
          "id": "c2",
          "name": "Bob Smith",
          "resume_summary": "6 years startup experience, built payment system..."
        },
        {
          "id": "c3",
          "name": "Carol Johnson",
          "resume_summary": "10 years enterprise, architect role..."
        }
      ],
      "ranking": ["c2", "c1", "c3"],
      "reasoning": "Bob's startup experience and hands-on approach to building systems from scratch makes him the strongest fit for our growth stage..."
    }
  ]
}
```

### 2. Run Batch Audit

```bash
curl -X POST https://council.example.com/api/audit/batch \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @decisions.json
```

### 3. Review Audit Report

```json
{
  "decision_id": "hire-2026-001",
  "audited_at": "2026-01-20T15:00:00Z",

  "ranking_agreement": {
    "council_ranking": ["c1", "c2", "c3"],
    "original_ranking": ["c2", "c1", "c3"],
    "kendall_tau": 0.67,
    "exact_match": false,
    "top_pick_match": false
  },

  "reasoning_analysis": {
    "reasoning_alignment": "partial",
    "missing_considerations": [
      "Alice's FAANG experience provides established best practices for scaling",
      "Leadership track record stronger for Alice (team of 12 vs Bob's individual contributor focus)"
    ],
    "questionable_assumptions": [
      "Assumption that startup experience is more valuable than enterprise scaling experience may not hold for a company past initial growth stage"
    ]
  },

  "flag_status": "review_recommended",
  "flag_reasons": [
    "Top candidate disagreement: Council chose Alice Chen vs original Bob Smith",
    "2 considerations not addressed in original reasoning"
  ]
}
```

---

## Use Cases

### 1. Periodic Quality Assurance
Run weekly/monthly audits on all decisions to track decision quality over time.

### 2. High-Stakes Decision Validation
Audit specific important decisions (executive hires, large contracts) before finalizing.

### 3. Process Refinement
Analyze patterns in disagreements to improve candidateaudit.com's evaluation criteria and prompts.

### 4. Bias Detection
Identify if certain types of candidates are consistently ranked differently by the council vs the primary system.

---

## Future Extensions

### Calibration Mode
Feed the council decisions with known outcomes to calibrate the flagging thresholds.

### Confidence Scoring
Train a model on audit results to predict which decisions are most likely to need review.

### Automated Feedback Loop
Optionally feed council insights back to candidateaudit.com to improve future decisions.

---

# Option B: Real-time API Integration

Instead of auditing decisions after the fact, candidateaudit.com calls LLM Council **during** its decision-making process — getting a "second opinion" before finalizing any ranking.

---

## Real-time API Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         candidateaudit.com                                   │
│                                                                              │
│  User submits candidates + criteria                                          │
│           │                                                                  │
│           ▼                                                                  │
│  ┌─────────────────┐                                                         │
│  │ Primary Decision │  (your existing logic)                                 │
│  │ Engine           │                                                         │
│  └────────┬────────┘                                                         │
│           │                                                                  │
│           │  ranking + reasoning                                             │
│           ▼                                                                  │
│  ┌─────────────────┐         ┌─────────────────────────────────────────┐    │
│  │ Council Gateway │────────►│         LLM Council API                 │    │
│  │ (async call)    │◄────────│  POST /api/validate                     │    │
│  └────────┬────────┘         │                                         │    │
│           │                  │  - Receives: candidates, criteria,      │    │
│           │                  │              primary ranking, reasoning │    │
│           │                  │  - Returns: council ranking, agreement  │    │
│           │                  │             score, flags, synthesis     │    │
│           ▼                  └─────────────────────────────────────────┘    │
│  ┌─────────────────┐                                                         │
│  │ Decision Router │                                                         │
│  │                 │                                                         │
│  │  if (agreed)    │──────► Finalize decision                               │
│  │  if (minor diff)│──────► Show warning, allow override                    │
│  │  if (major diff)│──────► Require human review                            │
│  └─────────────────┘                                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Real-time Validation API

### Endpoint

```typescript
POST /api/council/validate
Content-Type: application/json
Authorization: Bearer <token>

{
  // Context
  "job_description": "Senior Backend Engineer...",
  "evaluation_criteria": ["5+ years experience", "distributed systems", ...],

  // Candidates
  "candidates": [
    { "id": "c1", "name": "Alice", "summary": "..." },
    { "id": "c2", "name": "Bob", "summary": "..." },
    { "id": "c3", "name": "Carol", "summary": "..." }
  ],

  // Primary decision to validate
  "primary_ranking": ["c2", "c1", "c3"],
  "primary_reasoning": "Bob's startup experience...",

  // Options
  "options": {
    "model_tier": "budget",           // or "premium"
    "agreement_threshold": 0.7,       // Kendall tau threshold
    "return_full_deliberation": false // Include stage1/2/3 details?
  }
}
```

### Response

```typescript
{
  "request_id": "val-123",
  "validated_at": "2026-01-22T...",

  // Quick verdict
  "verdict": "review_recommended",  // "confirmed" | "review_recommended" | "significant_disagreement"

  // Agreement metrics
  "agreement": {
    "ranking_match": false,
    "top_pick_match": false,         // Most important - does #1 match?
    "kendall_tau": 0.67,             // Rank correlation
    "council_ranking": ["c1", "c2", "c3"],
    "primary_ranking": ["c2", "c1", "c3"]
  },

  // Reasoning comparison
  "reasoning_analysis": {
    "alignment": "partial",          // "aligned" | "partial" | "divergent"
    "council_reasoning": "Alice's FAANG experience provides...",
    "agreements": [
      "Both recognize strong technical skills across candidates"
    ],
    "disagreements": [
      "Council weighs leadership track record more heavily",
      "Primary decision may undervalue enterprise scaling experience"
    ],
    "missing_considerations": [
      "Alice led team of 12 - stronger leadership signal"
    ]
  },

  // Flags for UI
  "flags": [
    {
      "severity": "high",
      "message": "Top candidate disagreement: Council prefers Alice over Bob"
    },
    {
      "severity": "medium",
      "message": "2 evaluation criteria not addressed in primary reasoning"
    }
  ],

  // Optional: full deliberation data
  "deliberation": null  // or { stage1, stage2, stage3 } if requested
}
```

---

## Integration Patterns

### Pattern 1: Blocking Validation

Wait for Council validation before showing results to user:

```typescript
// In candidateaudit.com
async function makeDecision(candidates, criteria) {
  // Your primary logic
  const primaryResult = await yourDecisionEngine(candidates, criteria);

  // Validate with Council (blocks until response)
  const validation = await councilApi.validate({
    candidates,
    criteria,
    primary_ranking: primaryResult.ranking,
    primary_reasoning: primaryResult.reasoning
  });

  if (validation.verdict === 'confirmed') {
    return { ...primaryResult, confidence: 'high', validated: true };
  }

  if (validation.verdict === 'review_recommended') {
    return {
      ...primaryResult,
      confidence: 'medium',
      validated: false,
      council_alternative: validation.agreement.council_ranking,
      flags: validation.flags
    };
  }

  // Significant disagreement - require human review
  return {
    status: 'pending_review',
    primary: primaryResult,
    council: validation,
    requires_human: true
  };
}
```

### Pattern 2: Async Validation (Fire-and-forget)

Return primary result immediately, validate in background:

```typescript
// In candidateaudit.com
async function makeDecision(candidates, criteria) {
  const primaryResult = await yourDecisionEngine(candidates, criteria);

  // Fire off validation async - don't block
  councilApi.validateAsync({
    decision_id: primaryResult.id,
    candidates,
    criteria,
    primary_ranking: primaryResult.ranking,
    primary_reasoning: primaryResult.reasoning,
    callback_url: 'https://candidateaudit.com/webhooks/council'
  });

  // Return immediately with primary result
  return { ...primaryResult, validation_pending: true };
}

// Later, webhook receives validation result
app.post('/webhooks/council', (req) => {
  const { decision_id, validation } = req.body;

  // Update the decision record
  await db.decisions.update(decision_id, {
    council_validated: true,
    council_verdict: validation.verdict,
    council_flags: validation.flags
  });

  // Notify user if significant disagreement
  if (validation.verdict === 'significant_disagreement') {
    await notifyUser(decision_id, validation);
  }
});
```

### Pattern 3: Parallel Comparison

Run both systems in parallel, show user both perspectives:

```typescript
// Run both in parallel, show user both results
async function makeDecision(candidates, criteria) {
  const [primaryResult, councilResult] = await Promise.all([
    yourDecisionEngine(candidates, criteria),
    councilApi.query({ candidates, criteria })  // Full council deliberation
  ]);

  return {
    primary: primaryResult,
    council: councilResult,
    comparison: compareRankings(primaryResult.ranking, councilResult.ranking)
  };
}
```

---

## UI Integration Examples

### Option A: Confidence Badge

```
┌─────────────────────────────────────────┐
│  Recommended: Bob Smith                 │
│  ✓ Council Validated                    │  ← Green badge
│                                         │
│  #1 Bob Smith     ████████████ 92%      │
│  #2 Alice Chen    ████████░░░░ 85%      │
│  #3 Carol Johnson █████░░░░░░░ 71%      │
└─────────────────────────────────────────┘
```

### Option B: Disagreement Warning

```
┌─────────────────────────────────────────┐
│  Recommended: Bob Smith                 │
│  ⚠️ Council Disagrees                   │  ← Yellow warning
│                                         │
│  Your Ranking:        Council Ranking:  │
│  #1 Bob Smith         #1 Alice Chen     │
│  #2 Alice Chen        #2 Bob Smith      │
│  #3 Carol Johnson     #3 Carol Johnson  │
│                                         │
│  [View Council Reasoning] [Override]    │
└─────────────────────────────────────────┘
```

### Option C: Side-by-Side Deliberation

```
┌─────────────────────────────────────────────────────────────┐
│  ┌─────────────────────┐  ┌─────────────────────────────┐  │
│  │  Your Analysis      │  │  Council Analysis           │  │
│  │                     │  │                             │  │
│  │  Bob's startup      │  │  Alice's FAANG experience   │  │
│  │  experience makes   │  │  provides established best  │  │
│  │  him ideal for...   │  │  practices for scaling...   │  │
│  │                     │  │                             │  │
│  │  #1 Bob             │  │  #1 Alice                   │  │
│  │  #2 Alice           │  │  #2 Bob                     │  │
│  │  #3 Carol           │  │  #3 Carol                   │  │
│  └─────────────────────┘  └─────────────────────────────┘  │
│                                                             │
│  Agreement: 67% | Top Pick: ✗ Disagree | [Resolve]         │
└─────────────────────────────────────────────────────────────┘
```

---

## When to Use Real-time API

**Good fit if:**
- High-stakes decisions where you want validation before commitment
- Users expect to see multiple perspectives
- You want to build trust by showing "second opinion"
- Decision volume is low enough that latency is acceptable

**Not ideal if:**
- High throughput (many decisions/minute)
- Latency-sensitive UX
- You want to keep candidateaudit.com simple
- Budget constraints (every decision costs LLM tokens)

---

## Real-time API Database Schema

```sql
-- Validation requests (for async pattern)
CREATE TABLE validation_requests (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  decision_id TEXT NOT NULL,           -- ID from candidateaudit.com

  -- Input
  request_data TEXT NOT NULL,          -- JSON: validation request
  callback_url TEXT,                   -- Webhook URL for async

  -- Output
  status TEXT NOT NULL DEFAULT 'pending',  -- pending | processing | complete | failed
  verdict TEXT,
  response_data TEXT,                  -- JSON: full validation response

  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  completed_at TEXT
);

CREATE INDEX idx_validation_requests_user ON validation_requests(user_id);
CREATE INDEX idx_validation_requests_status ON validation_requests(status);
CREATE INDEX idx_validation_requests_decision ON validation_requests(decision_id);
```

---

## Related Documentation

- [LLM Council PRD](./PRD.md) — Main project requirements
- [Decision Making Process](./decision-making-process.md) — How the council reaches consensus
- [Quality Assurance](./quality-assurance.md) — Testing and monitoring approach
