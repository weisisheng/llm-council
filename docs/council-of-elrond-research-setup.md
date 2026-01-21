# Council of Elrond: Hosting, Architecture & Cost Research

> *"You have only one choice. The Ring must be destroyed."* - Elrond
>
> Research findings for optimizing the LLM Council deliberation system.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Current Architecture](#current-architecture)
3. [Hosting Recommendations](#1-hosting-recommendations)
4. [Cloudflare Workers Feasibility](#2-cloudflare-workers-feasibility)
5. [Hono + Bun Refactor Analysis](#3-hono--bun-refactor-analysis)
6. [Cost Optimization Strategies](#4-cost-optimization-strategies)
7. [Batch API Analysis](#5-batch-api-analysis)
8. [Monthly Cost Estimates](#6-monthly-cost-estimates)
9. [Migration Roadmap](#7-migration-roadmap)
10. [Recommendations Summary](#8-recommendations-summary)

---

## Executive Summary

| Question | Answer |
|----------|--------|
| Best hosting? | **Hetzner VPS** ($4-6/mo) for simplicity; **Cloudflare Workers** ($5-8/mo) for edge |
| Cloudflare viable? | **Yes** - 5-min CPU timeout on paid tier handles LLM calls |
| Hono+Bun worth it? | **Yes** for edge deployment, TypeScript monorepo benefits |
| Cheapest design? | Switch to budget models (80% savings in 5 minutes) |
| Batch API available? | OpenAI, Anthropic, Google - all 50% off, 24h max turnaround |
| Batch feasible? | **Hybrid approach** - real-time default, batch for bulk/reports |
| Cost per query? | $0.175 (premium) → $0.035 (budget models) |

---

## Current Architecture

### Tech Stack
- **Backend**: FastAPI (Python 3.10+) with async/await
- **Frontend**: React 19 + Vite (static SPA)
- **Storage**: JSON files in `data/conversations/`
- **LLM API**: OpenRouter (unified gateway)

### API Call Pattern Per User Query

```
User Query
    │
    ├─ Title Generation (async background)
    │   └─ 1 call to gemini-2.5-flash
    │
    └─ Stage 1: Collect Individual Responses
         ├─ Model 1 (gpt-5.1) ────────┐
         ├─ Model 2 (gemini-3-pro) ───┤ 4 parallel calls
         ├─ Model 3 (claude-sonnet) ──┤
         └─ Model 4 (grok-4) ─────────┘
              │
              ↓
         Stage 2: Peer Rankings (Anonymized)
         ├─ Model 1 evaluates A,B,C,D ─┐
         ├─ Model 2 evaluates A,B,C,D ─┤ 4 parallel calls
         ├─ Model 3 evaluates A,B,C,D ─┤
         └─ Model 4 evaluates A,B,C,D ─┘
              │
              ↓
         Stage 3: Chairman Synthesis
         └─ 1 call to gemini-3-pro
              │
              ↓
         Return Results + Metadata
```

**Total: 9 API calls per query** (4 + 4 + 1)

### Token Usage Per Query

| Stage | Input Tokens | Output Tokens | Total |
|-------|--------------|---------------|-------|
| Stage 1 (x4) | 2,000 | 6,000 | 8,000 |
| Stage 2 (x4) | 12,000 | 3,200 | 15,200 |
| Stage 3 (x1) | 8,000 | 2,000 | 10,000 |
| Title (x1) | 200 | 50 | 250 |
| **Total** | **22,200** | **11,250** | **~33,450** |

---

## 1. Hosting Recommendations

### Option A: Simple VPS (Recommended for Most Use Cases)

| Rank | Provider | Monthly Cost | Best For |
|------|----------|--------------|----------|
| 1 | **Hetzner CX22** | $4-6 | Best value, EU-based |
| 2 | DigitalOcean Droplet | $6 | US presence, great docs |
| 3 | DigitalOcean App Platform | $5-12 | Managed deployments |
| 4 | Fly.io | $5-20 | Global edge distribution |
| 5 | Railway | $5+ | Fast prototyping |

**Why VPS over Serverless:**
- 120-second LLM timeouts exceed most serverless limits
- Persistent file storage needed for conversations
- Simple deployment: `python -m backend.main` just works
- Predictable monthly cost

### Option B: Cloudflare Workers (Edge Deployment)

Now viable with paid tier's 5-minute CPU timeout. See detailed analysis below.

### Not Recommended

| Platform | Issue |
|----------|-------|
| AWS Lambda | Complex timeout configuration, cold starts |
| Vercel Functions | 60s limit too short |
| Netlify Functions | 10s limit way too short |

---

## 2. Cloudflare Workers Feasibility

### Verdict: **NOW VIABLE** on Paid Tier

#### Critical Insight: CPU Time vs Wall-Clock Time

| Metric | What It Measures | Your App's Usage |
|--------|------------------|------------------|
| **CPU Time** | Active processing | ~150-200ms per request |
| **Wall-Clock Time** | Total elapsed (including I/O waits) | Up to 120s per LLM call |

**Key**: When your Worker awaits an external API (OpenRouter), that wait time is **FREE** - it does not count against CPU limits. Your actual CPU usage per request is ~200ms, well under even the 30-second default limit.

#### Cloudflare Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloudflare Edge                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐   │
│   │   Pages     │    │   Workers   │    │     D1      │   │
│   │  (Frontend) │───▶│   (API)     │───▶│  (SQLite)   │   │
│   └─────────────┘    └──────┬──────┘    └─────────────┘   │
│                             │                              │
│                             ▼                              │
│                      ┌─────────────┐                       │
│                      │  OpenRouter │                       │
│                      │    API      │                       │
│                      └─────────────┘                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### Storage Options

| Storage | Use Case | Recommendation |
|---------|----------|----------------|
| **D1** | SQLite database | **BEST** - conversations + messages |
| **R2** | Object storage | Good for exports/backups |
| **KV** | Key-value cache | Works but D1 is better |

#### D1 Schema

```sql
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,
  created_at TEXT,
  title TEXT
);

CREATE TABLE messages (
  id INTEGER PRIMARY KEY,
  conversation_id TEXT,
  role TEXT,
  content TEXT,
  stage1 TEXT,  -- JSON blob
  stage2 TEXT,  -- JSON blob
  stage3 TEXT,  -- JSON blob
  FOREIGN KEY (conversation_id) REFERENCES conversations(id)
);
```

#### Cloudflare Pricing (Paid Plan)

| Component | Free Tier | Paid Rate | Estimated Monthly |
|-----------|-----------|-----------|-------------------|
| Base plan | - | $5/month | $5.00 |
| Requests | 100K/day | $0.30/M | ~$0.03 |
| CPU ms | 10M/day | $0.02/M ms | ~$0.40 |
| D1 Reads | 5M/day | $0.75/M | ~$0.38 |
| D1 Writes | 100K/day | $0.75/M | ~$0.04 |
| D1 Storage | 5GB | $0.75/GB/mo | ~$0.08 |
| **Total** | | | **~$6-8/mo** |

---

## 3. Hono + Bun Refactor Analysis

### Why Consider This Refactor?

| Benefit | Description |
|---------|-------------|
| **Edge-native** | Hono designed for Workers, <15KB bundle |
| **TypeScript monorepo** | Shared types between frontend/backend |
| **Fast cold starts** | 5-20ms vs 200-500ms for Python |
| **Bun performance** | Native fetch, excellent parallel HTTP |

### Performance Comparison

| Metric | FastAPI (Python) | Hono + Bun |
|--------|------------------|------------|
| Cold start | 200-500ms | 5-20ms |
| Request handling | ~1000 req/s | ~100,000 req/s |
| Memory footprint | 50-100MB | 10-20MB |
| Bundle size | N/A | <15KB |

**Note**: For this app, performance differences are negligible since LLM API latency (10-60s) dominates.

### Monorepo Structure

```
llm-council/
├── apps/
│   ├── web/                    # React frontend
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── api.ts
│   │   │   └── App.tsx
│   │   └── package.json
│   │
│   └── api/                    # Hono backend
│       ├── src/
│       │   ├── routes/
│       │   │   └── conversations.ts
│       │   ├── services/
│       │   │   ├── council.ts
│       │   │   ├── openrouter.ts
│       │   │   └── storage.ts
│       │   └── index.ts
│       └── wrangler.json
│
├── packages/
│   └── shared/                 # Shared types
│       └── src/types/
│
├── package.json
└── turbo.json
```

### Migration Effort

| Module | Python Lines | TypeScript Effort |
|--------|--------------|-------------------|
| config.py | 26 | 30 min |
| openrouter.py | 79 | 2 hours |
| council.py | 335 | 4 hours |
| storage.py | 172 | 3 hours (D1 adapter) |
| main.py | 200 | 2 hours |
| **Total** | **812** | **~4-5 days** |

### Code Translation Example

**Python (current):**
```python
async def query_models_parallel(models, messages):
    tasks = [query_model(model, messages) for model in models]
    responses = await asyncio.gather(*tasks)
    return {m: r for m, r in zip(models, responses) if r}
```

**TypeScript (Hono):**
```typescript
async function queryModelsParallel(
  models: string[],
  messages: Message[],
  env: Env
): Promise<Record<string, ModelResponse>> {
  const responses = await Promise.all(
    models.map(model => queryModel(model, messages, env))
  );
  return Object.fromEntries(
    models.map((m, i) => [m, responses[i]]).filter(([_, r]) => r)
  );
}
```

---

## 4. Cost Optimization Strategies

### A. Model Substitutions (Biggest Impact: 80% savings)

| Current Model | Cheaper Alternative | Cost Reduction |
|--------------|---------------------|----------------|
| GPT-5.1 ($1.25/$10) | DeepSeek V3 ($0.19/$0.87) | ~90% |
| Claude-sonnet-4.5 ($3/$15) | Claude Haiku 4.5 ($1/$5) | ~66% |
| Gemini-3-pro ($2/$12) | Gemini 2.5 Flash ($0.30/$2.50) | ~80% |
| Grok-4 ($1/$10) | Grok-4-fast ($1/$1) | ~90% output |

**Budget Council Config** (edit `backend/config.py`):
```python
COUNCIL_MODELS = [
    "deepseek/deepseek-chat-v3",
    "google/gemini-2.5-flash",
    "anthropic/claude-3.5-haiku",
    "x-ai/grok-4-fast",
]
CHAIRMAN_MODEL = "google/gemini-2.5-flash"
```

**Per-query cost: $0.175 → $0.035 (80% reduction)**

### B. Reduce Council Size

| Configuration | API Calls | Savings |
|---------------|-----------|---------|
| 4-model council (current) | 9 | baseline |
| 3-model council | 7 | 22% |
| 2-model council | 5 | 44% |

### C. Response Caching

- Cache identical queries for 24 hours
- Implementation: Redis, in-memory, or D1
- Savings: 10-50% depending on query repetition

### D. Prompt Compression

- Current prompts ~500-1000 tokens each
- Compress Stage 2 evaluation prompt
- Use system prompts for repeated instructions
- Savings: 20-30% input token reduction

---

## 5. Batch API Analysis

### Provider Support

| Provider | Batch API | Discount | Max Turnaround | Typical Completion |
|----------|-----------|----------|----------------|-------------------|
| **OpenAI** | Yes | 50% | 24 hours | 1-6 hours |
| **Anthropic** | Yes | 50% | 24 hours | 1-6 hours |
| **Google Gemini** | Yes | 50% | 24 hours | 1-6 hours |
| **xAI (Grok)** | No | - | - | - |
| **DeepSeek** | No | - | - | Use prompt caching |
| **OpenRouter** | **No** | - | - | No batch support |

### Key Limitation

**OpenRouter does NOT support batch APIs.** To use batch pricing, you must call provider APIs directly:
- `api.openai.com` for OpenAI
- `api.anthropic.com` for Anthropic
- `generativelanguage.googleapis.com` for Gemini

This loses OpenRouter's unified interface but gains 50% cost savings.

### Batch Feasibility for Council

#### Can 3-Stage Council Use Batch?

**Yes, with smart design:**

```
Submit Time 0:
├── Stage 1: 4 jobs (model responses)
└── Stage 2: 4 jobs (pre-computed with placeholder responses)

Wait for Stage 1 completion (~1-6 hours)

Submit Stage 2 with actual responses
Wait for Stage 2 completion (~1-6 hours)

Submit Stage 3 (or do real-time, it's just 1 call)

Total realistic wait: 2-12 hours
```

#### When Batch Makes Sense

| Use Case | Real-Time | Batch |
|----------|-----------|-------|
| Interactive Q&A | Yes | No |
| Weekly reports | No | Yes |
| Document batch analysis | No | Yes |
| Training data generation | No | Yes |
| Bulk research queries | Maybe | Yes |

#### Recommended: Hybrid Approach

```
User submits query
    │
    ├─ "Quick Mode" (default)
    │   └─ Real-time, full price, ~60s response
    │
    └─ "Deep Mode" (opt-in)
        └─ Batch API, 50% off, email when ready
```

---

## 6. Monthly Cost Estimates

### Per-Query Cost Breakdown

**Current Premium Models:**

| Stage | Models | Input Cost | Output Cost | Total |
|-------|--------|------------|-------------|-------|
| Stage 1 | 4x council | $0.0036 | $0.0705 | $0.074 |
| Stage 2 | 4x council | $0.022 | $0.038 | $0.060 |
| Stage 3 | 1x chairman | $0.016 | $0.024 | $0.040 |
| Title | 1x flash | ~$0 | ~$0 | ~$0 |
| **Total** | | | | **$0.175** |

**Budget Models:**

| Stage | Models | Total |
|-------|--------|-------|
| All stages | budget council | **$0.035** |

### Monthly Projections

#### Premium Models ($0.175/query)

| Usage Tier | Queries/Day | Queries/Month | LLM Cost | Hosting | **Total** |
|------------|-------------|---------------|----------|---------|-----------|
| Personal | 10 | 300 | $52 | $4-6 | **$56-58/mo** |
| Light Team | 50 | 1,500 | $263 | $6 | **$269/mo** |
| Heavy Team | 200 | 6,000 | $1,050 | $12 | **$1,062/mo** |
| Production | 1,000 | 30,000 | $5,250 | $50 | **$5,300/mo** |

#### Budget Models ($0.035/query)

| Usage Tier | Queries/Month | LLM Cost | Hosting | **Total** |
|------------|---------------|----------|---------|-----------|
| Personal | 300 | $11 | $4-6 | **$15-17/mo** |
| Light Team | 1,500 | $53 | $6 | **$59/mo** |
| Heavy Team | 6,000 | $210 | $12 | **$222/mo** |
| Production | 30,000 | $1,050 | $50 | **$1,100/mo** |

#### With Batch API (50% off LLM costs)

| Usage Tier | Premium + Batch | Budget + Batch |
|------------|-----------------|----------------|
| Personal | $30/mo | $10/mo |
| Light Team | $137/mo | $33/mo |
| Heavy Team | $537/mo | $117/mo |
| Production | $2,675/mo | $575/mo |

### Cost Optimization Summary

| Strategy | Effort | Savings |
|----------|--------|---------|
| Switch to budget models | 5 minutes | **80%** |
| 3-model council | 30 minutes | 22% |
| Add caching | 1-2 days | 10-50% |
| Batch API | 5-10 days | 50% |
| All combined | 1-2 weeks | **90%+** |

---

## 7. Migration Roadmap

### Option A: Stay with Python (Low Effort)

1. **Day 1**: Switch to budget models (edit config.py)
2. **Day 2-3**: Add response caching
3. **Day 4-5**: Deploy to Hetzner VPS with Docker
4. **Done**: $15-60/month depending on usage

### Option B: Cloudflare Workers + Hono (Medium Effort)

1. **Day 1**: Set up monorepo structure, shared types
2. **Day 2**: Port openrouter.ts and config
3. **Day 3**: Port council.ts (3-stage logic)
4. **Day 4**: Create D1 storage adapter
5. **Day 5**: Port Hono routes, deploy to Workers
6. **Day 6**: Update frontend, test end-to-end
7. **Done**: $6-8/month base + usage

### Option C: Full Optimization (High Effort)

1. **Week 1**: Hono + Bun migration to Cloudflare
2. **Week 2**: Implement batch API support (direct provider calls)
3. **Week 3**: Add hybrid real-time/batch mode
4. **Week 4**: Add caching, monitoring, cost tracking
5. **Done**: Maximum cost efficiency, ~$10-100/month at any scale

---

## 8. Recommendations Summary

### For Personal/Hobby Use

| Recommendation | Details |
|----------------|---------|
| **Hosting** | Hetzner VPS ($4/mo) |
| **Models** | Budget council (config change) |
| **Architecture** | Keep current Python/FastAPI |
| **Expected cost** | $15-60/month |

### For Team/Startup

| Recommendation | Details |
|----------------|---------|
| **Hosting** | Cloudflare Workers + D1 |
| **Models** | Mix of premium + budget |
| **Architecture** | Hono + Bun refactor |
| **Expected cost** | $60-270/month |

### For Production SaaS

| Recommendation | Details |
|----------------|---------|
| **Hosting** | Cloudflare Workers (global edge) |
| **Models** | Configurable per-user tier |
| **Architecture** | Full Hono + Bun with batch mode |
| **Features** | Usage limits, billing, batch queue |
| **Expected cost** | $500-2,000/month at scale |

---

## Quick Reference: Files to Modify

| Change | File | Effort |
|--------|------|--------|
| Swap models | `backend/config.py` | 5 min |
| Reduce council size | `backend/config.py` | 5 min |
| Add caching | `backend/council.py` | 1-2 days |
| Database migration | `backend/storage.py` | 2-3 days |
| Batch API support | New module + queue system | 5-10 days |
| Full Hono refactor | All backend files | 4-5 days |

---

## Sources

- [Cloudflare Workers Pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Cloudflare Workers 5-minute CPU Limits](https://developers.cloudflare.com/changelog/2025-03-25-higher-cpu-limits/)
- [Cloudflare D1 Pricing](https://developers.cloudflare.com/d1/platform/pricing/)
- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Google Gemini Batch API](https://cloud.google.com/vertex-ai/docs/generative-ai/batch-prediction)
- [Hono Documentation](https://hono.dev/docs/)
- [OpenRouter Pricing](https://openrouter.ai/pricing)

---

## 9. Feature Flags System Design

### Overview

A user-controllable feature flag system enabling runtime configuration of processing mode, model selection, and council behavior.

### Core Feature Flags

| Flag | Type | Values | Default |
|------|------|--------|---------|
| `processing_mode` | enum | `"realtime"`, `"batch"` | `"realtime"` |
| `model_tier` | enum | `"premium"`, `"budget"`, `"custom"` | `"premium"` |
| `council_size` | int | 2-5 | 4 |
| `skip_stage2` | bool | true/false | false |
| `response_length` | enum | `"concise"`, `"standard"`, `"detailed"` | `"standard"` |
| `anonymize_stage2` | bool | true/false | true |
| `chairman_model` | string | model ID or `"auto"` | `"auto"` |
| `enable_caching` | bool | true/false | true |

### Model Tier Configurations

**Premium Tier:**
```typescript
const PREMIUM_COUNCIL = [
  "openai/gpt-5.1",
  "anthropic/claude-sonnet-4.5",
  "google/gemini-3-pro-preview",
  "x-ai/grok-4",
];
```

**Budget Tier:**
```typescript
const BUDGET_COUNCIL = [
  "deepseek/deepseek-v3",
  "anthropic/claude-3.5-haiku",
  "google/gemini-2.5-flash",
  "x-ai/grok-fast",
];
```

### API Schema

```typescript
interface FeatureFlags {
  processing_mode: "realtime" | "batch";
  model_tier: "premium" | "budget" | "custom";
  council_size: number;  // 2-5
  skip_stage2: boolean;
  response_length: "concise" | "standard" | "detailed";
  anonymize_stage2: boolean;
  chairman_model: string | null;
  enable_caching: boolean;
  custom_council_models?: string[];
  custom_chairman_model?: string;
}

// Request with flags
interface SendMessageRequest {
  content: string;
  flags?: Partial<FeatureFlags>;
}
```

### Cost Impact by Flag

| Flag Change | API Calls | Cost Impact |
|-------------|-----------|-------------|
| Premium → Budget | Same | **-80%** |
| 4 models → 3 models | 9 → 7 | **-22%** |
| 4 models → 2 models | 9 → 5 | **-44%** |
| Skip Stage 2 | 9 → 5 | **-44%** |
| Realtime → Batch | Same | **-50%** |

---

## 10. Migration Guide: Python → Hono + Bun + Vite + Preact

### Why Preact Over React?

| Aspect | React 19 | Preact |
|--------|----------|--------|
| Bundle size | ~40KB | **3KB** |
| Compatibility | Native | React-compatible via `preact/compat` |
| Performance | Fast | Slightly faster |
| Best for | Complex apps | Edge deployment, small bundles |

For Cloudflare Pages, Preact's 3KB bundle is ideal.

### Target Architecture

```
llm-council-v2/
├── apps/
│   ├── web/                      # Preact + Vite frontend
│   │   ├── src/
│   │   │   ├── components/
│   │   │   │   ├── ChatInterface.tsx
│   │   │   │   ├── Sidebar.tsx
│   │   │   │   ├── Stage1.tsx
│   │   │   │   ├── Stage2.tsx
│   │   │   │   ├── Stage3.tsx
│   │   │   │   └── SettingsPanel.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useConversation.ts
│   │   │   ├── api.ts
│   │   │   ├── app.tsx
│   │   │   └── main.tsx
│   │   ├── index.html
│   │   ├── vite.config.ts
│   │   └── package.json
│   │
│   └── api/                      # Hono backend
│       ├── src/
│       │   ├── routes/
│       │   │   ├── conversations.ts
│       │   │   └── batch.ts
│       │   ├── services/
│       │   │   ├── council.ts
│       │   │   ├── openrouter.ts
│       │   │   └── storage.ts
│       │   ├── types/
│       │   │   └── index.ts
│       │   └── index.ts
│       ├── wrangler.toml
│       └── package.json
│
├── packages/
│   └── shared/                   # Shared types
│       ├── src/
│       │   ├── types.ts
│       │   └── index.ts
│       └── package.json
│
├── package.json                  # Workspace root
├── bun.lockb
└── turbo.json
```

### Step-by-Step Migration

#### Phase 1: Project Setup (Day 1)

**1.1 Initialize Monorepo**

```bash
mkdir llm-council-v2 && cd llm-council-v2
bun init -y

# Create workspace structure
mkdir -p apps/web apps/api packages/shared
```

**1.2 Root package.json**

```json
{
  "name": "llm-council",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "deploy": "turbo run deploy",
    "typecheck": "turbo run typecheck"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "typescript": "^5.4.0"
  }
}
```

**1.3 turbo.json**

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "deploy": {
      "dependsOn": ["build"]
    },
    "typecheck": {
      "dependsOn": ["^typecheck"]
    }
  }
}
```

**1.4 Shared Types Package**

```bash
cd packages/shared
bun init -y
```

`packages/shared/src/types.ts`:
```typescript
// Conversation types
export interface Conversation {
  id: string;
  created_at: string;
  title: string;
  messages: Message[];
}

export interface Message {
  role: "user" | "assistant";
  content?: string;
  stage1?: Stage1Result[];
  stage2?: Stage2Result[];
  stage3?: Stage3Result;
}

// Council types
export interface Stage1Result {
  model: string;
  response: string;
}

export interface Stage2Result {
  model: string;
  ranking: string;
  parsed_ranking: string[];
}

export interface Stage3Result {
  model: string;
  response: string;
}

export interface CouncilMetadata {
  label_to_model: Record<string, string>;
  aggregate_rankings: AggregateRanking[];
}

export interface AggregateRanking {
  model: string;
  average_rank: number;
  rankings_count: number;
}

// Feature flags
export interface FeatureFlags {
  processing_mode: "realtime" | "batch";
  model_tier: "premium" | "budget" | "custom";
  council_size: number;
  skip_stage2: boolean;
  response_length: "concise" | "standard" | "detailed";
  anonymize_stage2: boolean;
  chairman_model: string | null;
  enable_caching: boolean;
  custom_council_models?: string[];
  custom_chairman_model?: string;
}

// API types
export interface SendMessageRequest {
  content: string;
  flags?: Partial<FeatureFlags>;
}

export interface CouncilResponse {
  stage1: Stage1Result[];
  stage2: Stage2Result[];
  stage3: Stage3Result;
  metadata: CouncilMetadata;
}
```

#### Phase 2: Backend Migration (Days 2-3)

**2.1 Initialize Hono API**

```bash
cd apps/api
bun init -y
bun add hono
bun add -d wrangler @cloudflare/workers-types
```

`apps/api/package.json`:
```json
{
  "name": "@llm-council/api",
  "scripts": {
    "dev": "wrangler dev src/index.ts",
    "deploy": "wrangler deploy",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "hono": "^4.0.0",
    "@llm-council/shared": "workspace:*"
  },
  "devDependencies": {
    "wrangler": "^3.0.0",
    "@cloudflare/workers-types": "^4.0.0"
  }
}
```

**2.2 wrangler.toml**

```toml
name = "llm-council-api"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[vars]
ENVIRONMENT = "production"

[[d1_databases]]
binding = "DB"
database_name = "llm-council"
database_id = "your-database-id"
```

**2.3 Hono Entry Point**

`apps/api/src/index.ts`:
```typescript
import { Hono } from "hono";
import { cors } from "hono/cors";
import { conversationsRouter } from "./routes/conversations";
import { batchRouter } from "./routes/batch";

type Bindings = {
  DB: D1Database;
  OPENROUTER_API_KEY: string;
};

const app = new Hono<{ Bindings: Bindings }>();

// CORS middleware
app.use("/*", cors({
  origin: ["http://localhost:5173", "https://your-domain.pages.dev"],
}));

// Health check
app.get("/", (c) => c.json({ status: "ok" }));

// Routes
app.route("/api/conversations", conversationsRouter);
app.route("/api/batch", batchRouter);

export default app;
```

**2.4 OpenRouter Service**

`apps/api/src/services/openrouter.ts`:
```typescript
import type { Bindings } from "../types";

const OPENROUTER_API_URL = "https://openrouter.ai/api/v1/chat/completions";

interface Message {
  role: "user" | "assistant" | "system";
  content: string;
}

interface ModelResponse {
  content: string;
  reasoning_details?: string;
}

export async function queryModel(
  model: string,
  messages: Message[],
  env: Bindings,
  timeout = 120000
): Promise<ModelResponse | null> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(OPENROUTER_API_URL, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${env.OPENROUTER_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model,
        messages,
      }),
      signal: controller.signal,
    });

    if (!response.ok) {
      console.error(`Model ${model} returned ${response.status}`);
      return null;
    }

    const data = await response.json() as any;
    return {
      content: data.choices?.[0]?.message?.content ?? "",
      reasoning_details: data.choices?.[0]?.message?.reasoning_details,
    };
  } catch (error) {
    console.error(`Error querying ${model}:`, error);
    return null;
  } finally {
    clearTimeout(timeoutId);
  }
}

export async function queryModelsParallel(
  models: string[],
  messages: Message[],
  env: Bindings
): Promise<Record<string, ModelResponse>> {
  const results = await Promise.all(
    models.map((model) => queryModel(model, messages, env))
  );

  return Object.fromEntries(
    models
      .map((model, i) => [model, results[i]] as const)
      .filter(([_, response]) => response !== null)
  ) as Record<string, ModelResponse>;
}
```

**2.5 Council Service**

`apps/api/src/services/council.ts`:
```typescript
import type { FeatureFlags, Stage1Result, Stage2Result, Stage3Result, CouncilMetadata } from "@llm-council/shared";
import { queryModel, queryModelsParallel } from "./openrouter";
import type { Bindings } from "../types";

const TIERS = {
  premium: {
    council: [
      "openai/gpt-5.1",
      "anthropic/claude-sonnet-4.5",
      "google/gemini-3-pro-preview",
      "x-ai/grok-4",
    ],
    chairman: "google/gemini-3-pro-preview",
  },
  budget: {
    council: [
      "deepseek/deepseek-v3",
      "anthropic/claude-3.5-haiku",
      "google/gemini-2.5-flash",
      "x-ai/grok-fast",
    ],
    chairman: "google/gemini-2.5-flash",
  },
};

function resolveModels(flags: Partial<FeatureFlags>) {
  const tier = flags.model_tier ?? "premium";
  const size = flags.council_size ?? 4;

  if (tier === "custom" && flags.custom_council_models) {
    return {
      council: flags.custom_council_models.slice(0, size),
      chairman: flags.custom_chairman_model ?? TIERS.premium.chairman,
    };
  }

  const tierConfig = TIERS[tier as keyof typeof TIERS] ?? TIERS.premium;
  return {
    council: tierConfig.council.slice(0, size),
    chairman: flags.chairman_model ?? tierConfig.chairman,
  };
}

function getLengthModifier(length: string): string {
  if (length === "concise") {
    return "\n\nIMPORTANT: Be brief and direct (1-2 paragraphs max).";
  }
  if (length === "detailed") {
    return "\n\nIMPORTANT: Provide a comprehensive answer with examples.";
  }
  return "";
}

export async function runCouncil(
  userQuery: string,
  flags: Partial<FeatureFlags>,
  env: Bindings
): Promise<{
  stage1: Stage1Result[];
  stage2: Stage2Result[];
  stage3: Stage3Result;
  metadata: CouncilMetadata;
}> {
  const { council, chairman } = resolveModels(flags);
  const lengthMod = getLengthModifier(flags.response_length ?? "standard");

  // Stage 1: Collect responses
  const stage1Query = userQuery + lengthMod;
  const stage1Responses = await queryModelsParallel(
    council,
    [{ role: "user", content: stage1Query }],
    env
  );

  const stage1: Stage1Result[] = Object.entries(stage1Responses).map(
    ([model, response]) => ({
      model,
      response: response.content,
    })
  );

  if (stage1.length === 0) {
    return {
      stage1: [],
      stage2: [],
      stage3: { model: "error", response: "All models failed." },
      metadata: { label_to_model: {}, aggregate_rankings: [] },
    };
  }

  // Stage 2: Peer rankings (if not skipped)
  let stage2: Stage2Result[] = [];
  let labelToModel: Record<string, string> = {};
  let aggregateRankings: CouncilMetadata["aggregate_rankings"] = [];

  if (!flags.skip_stage2) {
    // Anonymize responses
    const labels = ["A", "B", "C", "D", "E"].slice(0, stage1.length);
    labelToModel = Object.fromEntries(
      stage1.map((r, i) => [`Response ${labels[i]}`, r.model])
    );

    const anonymizedResponses = stage1
      .map((r, i) => `**Response ${labels[i]}:**\n${r.response}`)
      .join("\n\n---\n\n");

    const rankingPrompt = `You are evaluating responses to: "${userQuery}"

Here are the responses:

${anonymizedResponses}

Evaluate each response for accuracy, completeness, and insight.
End with "FINAL RANKING:" followed by a numbered list (1 = best).`;

    const stage2Responses = await queryModelsParallel(
      council,
      [{ role: "user", content: rankingPrompt }],
      env
    );

    stage2 = Object.entries(stage2Responses).map(([model, response]) => ({
      model,
      ranking: response.content,
      parsed_ranking: parseRanking(response.content),
    }));

    aggregateRankings = calculateAggregateRankings(stage2, labelToModel);
  }

  // Stage 3: Chairman synthesis
  const synthesisContext = stage1
    .map((r) => `**${r.model}:**\n${r.response}`)
    .join("\n\n---\n\n");

  const rankingsContext = stage2.length > 0
    ? `\n\nPeer Rankings:\n${stage2.map((r) => `${r.model}: ${r.parsed_ranking.join(" > ")}`).join("\n")}`
    : "";

  const synthesisPrompt = `Original question: "${userQuery}"

Individual responses:
${synthesisContext}
${rankingsContext}

Synthesize the best answer, incorporating insights from all responses.`;

  const stage3Response = await queryModel(
    chairman,
    [{ role: "user", content: synthesisPrompt }],
    env
  );

  return {
    stage1,
    stage2,
    stage3: {
      model: chairman,
      response: stage3Response?.content ?? "Synthesis failed.",
    },
    metadata: {
      label_to_model: labelToModel,
      aggregate_rankings: aggregateRankings,
    },
  };
}

function parseRanking(text: string): string[] {
  const match = text.match(/FINAL RANKING:[\s\S]*$/i);
  if (!match) return [];

  const rankings: string[] = [];
  const lines = match[0].split("\n");
  for (const line of lines) {
    const responseMatch = line.match(/Response\s+([A-E])/i);
    if (responseMatch) {
      rankings.push(`Response ${responseMatch[1].toUpperCase()}`);
    }
  }
  return rankings;
}

function calculateAggregateRankings(
  stage2: Stage2Result[],
  labelToModel: Record<string, string>
): CouncilMetadata["aggregate_rankings"] {
  const scores: Record<string, { total: number; count: number }> = {};

  for (const result of stage2) {
    result.parsed_ranking.forEach((label, index) => {
      const model = labelToModel[label];
      if (model) {
        if (!scores[model]) scores[model] = { total: 0, count: 0 };
        scores[model].total += index + 1;
        scores[model].count += 1;
      }
    });
  }

  return Object.entries(scores)
    .map(([model, { total, count }]) => ({
      model,
      average_rank: total / count,
      rankings_count: count,
    }))
    .sort((a, b) => a.average_rank - b.average_rank);
}
```

**2.6 D1 Storage Service**

`apps/api/src/services/storage.ts`:
```typescript
import type { Conversation, Message } from "@llm-council/shared";

export class Storage {
  constructor(private db: D1Database) {}

  async listConversations(): Promise<Conversation[]> {
    const result = await this.db
      .prepare("SELECT * FROM conversations ORDER BY created_at DESC")
      .all();
    return result.results as Conversation[];
  }

  async getConversation(id: string): Promise<Conversation | null> {
    const conv = await this.db
      .prepare("SELECT * FROM conversations WHERE id = ?")
      .bind(id)
      .first();

    if (!conv) return null;

    const messages = await this.db
      .prepare("SELECT * FROM messages WHERE conversation_id = ? ORDER BY id")
      .bind(id)
      .all();

    return {
      ...(conv as any),
      messages: (messages.results as any[]).map((m) => ({
        role: m.role,
        content: m.content,
        stage1: m.stage1 ? JSON.parse(m.stage1) : undefined,
        stage2: m.stage2 ? JSON.parse(m.stage2) : undefined,
        stage3: m.stage3 ? JSON.parse(m.stage3) : undefined,
      })),
    };
  }

  async createConversation(): Promise<Conversation> {
    const id = crypto.randomUUID();
    const created_at = new Date().toISOString();

    await this.db
      .prepare("INSERT INTO conversations (id, created_at, title) VALUES (?, ?, ?)")
      .bind(id, created_at, "New Conversation")
      .run();

    return { id, created_at, title: "New Conversation", messages: [] };
  }

  async addMessage(conversationId: string, message: Message): Promise<void> {
    await this.db
      .prepare(`
        INSERT INTO messages (conversation_id, role, content, stage1, stage2, stage3)
        VALUES (?, ?, ?, ?, ?, ?)
      `)
      .bind(
        conversationId,
        message.role,
        message.content ?? null,
        message.stage1 ? JSON.stringify(message.stage1) : null,
        message.stage2 ? JSON.stringify(message.stage2) : null,
        message.stage3 ? JSON.stringify(message.stage3) : null
      )
      .run();
  }

  async updateTitle(conversationId: string, title: string): Promise<void> {
    await this.db
      .prepare("UPDATE conversations SET title = ? WHERE id = ?")
      .bind(title, conversationId)
      .run();
  }
}
```

**2.7 Conversations Router**

`apps/api/src/routes/conversations.ts`:
```typescript
import { Hono } from "hono";
import { Storage } from "../services/storage";
import { runCouncil } from "../services/council";
import type { SendMessageRequest } from "@llm-council/shared";

type Bindings = {
  DB: D1Database;
  OPENROUTER_API_KEY: string;
};

export const conversationsRouter = new Hono<{ Bindings: Bindings }>();

// List conversations
conversationsRouter.get("/", async (c) => {
  const storage = new Storage(c.env.DB);
  const conversations = await storage.listConversations();
  return c.json(conversations);
});

// Create conversation
conversationsRouter.post("/", async (c) => {
  const storage = new Storage(c.env.DB);
  const conversation = await storage.createConversation();
  return c.json(conversation);
});

// Get conversation
conversationsRouter.get("/:id", async (c) => {
  const storage = new Storage(c.env.DB);
  const conversation = await storage.getConversation(c.req.param("id"));
  if (!conversation) {
    return c.json({ error: "Not found" }, 404);
  }
  return c.json(conversation);
});

// Send message
conversationsRouter.post("/:id/message", async (c) => {
  const storage = new Storage(c.env.DB);
  const conversationId = c.req.param("id");
  const body = await c.req.json<SendMessageRequest>();

  // Save user message
  await storage.addMessage(conversationId, {
    role: "user",
    content: body.content,
  });

  // Run council
  const result = await runCouncil(body.content, body.flags ?? {}, c.env);

  // Save assistant message
  await storage.addMessage(conversationId, {
    role: "assistant",
    stage1: result.stage1,
    stage2: result.stage2,
    stage3: result.stage3,
  });

  return c.json(result);
});
```

#### Phase 3: Frontend Migration (Days 4-5)

**3.1 Initialize Preact + Vite**

```bash
cd apps/web
bun create vite . --template preact-ts
bun add @llm-council/shared
bun add preact-markdown  # or use marked
```

**3.2 Vite Config for Preact**

`apps/web/vite.config.ts`:
```typescript
import { defineConfig } from "vite";
import preact from "@preact/preset-vite";

export default defineConfig({
  plugins: [preact()],
  resolve: {
    alias: {
      react: "preact/compat",
      "react-dom": "preact/compat",
    },
  },
});
```

**3.3 API Client**

`apps/web/src/api.ts`:
```typescript
import type {
  Conversation,
  SendMessageRequest,
  CouncilResponse,
  FeatureFlags,
} from "@llm-council/shared";

const API_BASE = import.meta.env.VITE_API_URL || "http://localhost:8787";

export const api = {
  async listConversations(): Promise<Conversation[]> {
    const res = await fetch(`${API_BASE}/api/conversations`);
    return res.json();
  },

  async createConversation(): Promise<Conversation> {
    const res = await fetch(`${API_BASE}/api/conversations`, {
      method: "POST",
    });
    return res.json();
  },

  async getConversation(id: string): Promise<Conversation> {
    const res = await fetch(`${API_BASE}/api/conversations/${id}`);
    return res.json();
  },

  async sendMessage(
    conversationId: string,
    content: string,
    flags?: Partial<FeatureFlags>
  ): Promise<CouncilResponse> {
    const res = await fetch(
      `${API_BASE}/api/conversations/${conversationId}/message`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ content, flags } as SendMessageRequest),
      }
    );
    return res.json();
  },
};
```

**3.4 Main App Component**

`apps/web/src/app.tsx`:
```tsx
import { useState, useEffect } from "preact/hooks";
import type { Conversation, FeatureFlags, CouncilResponse } from "@llm-council/shared";
import { api } from "./api";
import { Sidebar } from "./components/Sidebar";
import { ChatInterface } from "./components/ChatInterface";
import { SettingsPanel } from "./components/SettingsPanel";
import "./app.css";

export function App() {
  const [conversations, setConversations] = useState<Conversation[]>([]);
  const [currentConversation, setCurrentConversation] = useState<Conversation | null>(null);
  const [loading, setLoading] = useState(false);
  const [flags, setFlags] = useState<Partial<FeatureFlags>>({
    model_tier: "premium",
    council_size: 4,
    skip_stage2: false,
  });

  useEffect(() => {
    api.listConversations().then(setConversations);
  }, []);

  const handleNewConversation = async () => {
    const conv = await api.createConversation();
    setConversations([conv, ...conversations]);
    setCurrentConversation(conv);
  };

  const handleSelectConversation = async (id: string) => {
    const conv = await api.getConversation(id);
    setCurrentConversation(conv);
  };

  const handleSendMessage = async (content: string) => {
    if (!currentConversation) return;

    setLoading(true);
    try {
      const result = await api.sendMessage(currentConversation.id, content, flags);

      // Refresh conversation to get updated messages
      const updated = await api.getConversation(currentConversation.id);
      setCurrentConversation(updated);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div class="app">
      <Sidebar
        conversations={conversations}
        currentId={currentConversation?.id}
        onNew={handleNewConversation}
        onSelect={handleSelectConversation}
      />
      <main class="main">
        <SettingsPanel flags={flags} onChange={setFlags} />
        <ChatInterface
          conversation={currentConversation}
          loading={loading}
          onSendMessage={handleSendMessage}
        />
      </main>
    </div>
  );
}
```

**3.5 Key Differences: React → Preact**

| React | Preact |
|-------|--------|
| `import { useState } from 'react'` | `import { useState } from 'preact/hooks'` |
| `className="..."` | `class="..."` |
| `ReactMarkdown` | Use `marked` + `dangerouslySetInnerHTML` or `preact-markdown` |
| `React.FC<Props>` | `FunctionComponent<Props>` |

#### Phase 4: Deployment (Day 6)

**4.1 D1 Database Setup**

```bash
# Create D1 database
wrangler d1 create llm-council

# Apply schema
wrangler d1 execute llm-council --file=./schema.sql
```

`schema.sql`:
```sql
CREATE TABLE IF NOT EXISTS conversations (
  id TEXT PRIMARY KEY,
  created_at TEXT NOT NULL,
  title TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  conversation_id TEXT NOT NULL,
  role TEXT NOT NULL,
  content TEXT,
  stage1 TEXT,
  stage2 TEXT,
  stage3 TEXT,
  FOREIGN KEY (conversation_id) REFERENCES conversations(id)
);

CREATE INDEX idx_messages_conversation ON messages(conversation_id);
```

**4.2 Deploy API to Workers**

```bash
cd apps/api

# Set secret
wrangler secret put OPENROUTER_API_KEY

# Deploy
wrangler deploy
```

**4.3 Deploy Frontend to Pages**

```bash
cd apps/web

# Build
bun run build

# Deploy
wrangler pages deploy dist --project-name=llm-council
```

**4.4 GitHub Actions CI/CD**

`.github/workflows/deploy.yml`:
```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1

      - name: Install dependencies
        run: bun install

      - name: Build
        run: bun run build

      - name: Deploy API
        run: bunx wrangler deploy
        working-directory: apps/api
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}

      - name: Deploy Frontend
        run: bunx wrangler pages deploy dist --project-name=llm-council
        working-directory: apps/web
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

### Migration Checklist

- [ ] **Day 1**: Monorepo setup, shared types package
- [ ] **Day 2**: Port `openrouter.ts`, `config.ts`
- [ ] **Day 3**: Port `council.ts` (3-stage logic)
- [ ] **Day 4**: Create D1 storage adapter, Hono routes
- [ ] **Day 5**: Migrate frontend to Preact, update API client
- [ ] **Day 6**: Deploy to Cloudflare, set up CI/CD
- [ ] **Day 7**: Testing, monitoring, documentation

### Cost After Migration

| Component | Before (VPS) | After (Cloudflare) |
|-----------|--------------|-------------------|
| Hosting | $4-6/mo | $5/mo base |
| Database | JSON files | D1 included |
| CDN | N/A | Included (Pages) |
| Global edge | No | Yes |
| **Total** | $4-6/mo | **$5-8/mo** |

---

## Sources

- [Cloudflare Workers Pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Cloudflare Workers 5-minute CPU Limits](https://developers.cloudflare.com/changelog/2025-03-25-higher-cpu-limits/)
- [Cloudflare D1 Pricing](https://developers.cloudflare.com/d1/platform/pricing/)
- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Google Gemini Batch API](https://cloud.google.com/vertex-ai/docs/generative-ai/batch-prediction)
- [Hono Documentation](https://hono.dev/docs/)
- [Preact Documentation](https://preactjs.com/guide/v10/getting-started)
- [OpenRouter Pricing](https://openrouter.ai/pricing)

---

## 11. Batch API Async Response Handling

### Design Options Comparison

| Option | Cost | Robustness | Complexity |
|--------|------|------------|------------|
| Cloudflare Cron (15 min) | Free | Good | Low |
| Cloudflare Queues + Backoff | Free | Excellent | Medium |
| Webhooks | Free | Best | High (limited provider support) |

### Recommended: Cron + Queues Hybrid

```
┌─────────────────────────────────────────────────────────────────┐
│                         Batch Job Flow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. User submits batch query                                    │
│     └─> Store job in D1 (status: "pending")                    │
│     └─> Submit to provider batch APIs                          │
│     └─> Update status to "processing"                          │
│     └─> Queue first check (5 min delay)                        │
│                                                                 │
│  2. Queue worker checks status                                  │
│     ├─> If complete: fetch results, store, email user          │
│     ├─> If failed: mark failed, email user                     │
│     └─> If processing: requeue with exponential backoff        │
│                                                                 │
│  3. Cron (every 15 min) as safety net                          │
│     └─> Check any jobs not checked in 30+ min                  │
│     └─> Handles edge cases (queue failures, etc.)              │
│                                                                 │
│  4. User can poll /api/batch/jobs/{id} anytime                 │
│     └─> Returns current status from D1                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Webhook Support Reality

| Provider | Webhook Support |
|----------|-----------------|
| OpenAI Batch | **No** - polling only |
| Anthropic Batch | **No** - polling only |
| Google Vertex | **Yes** - via Pub/Sub |

Since OpenAI and Anthropic don't support webhooks, **polling is required**.

### Exponential Backoff Schedule

```typescript
function getBackoffDelay(attempt: number): number {
  // 5min, 10min, 15min, 30min, 30min, 60min, 60min...
  const schedule = [300, 600, 900, 1800, 1800, 3600];
  return schedule[Math.min(attempt - 1, schedule.length - 1)];
}
```

| Attempt | Delay | Cumulative Wait |
|---------|-------|-----------------|
| 1 | 5 min | 5 min |
| 2 | 10 min | 15 min |
| 3 | 15 min | 30 min |
| 4 | 30 min | 1 hr |
| 5 | 30 min | 1.5 hr |
| 6+ | 60 min | 2.5+ hr |

### Cloudflare Cron Configuration

```toml
# wrangler.toml
[triggers]
crons = ["*/15 * * * *"]  # Every 15 minutes
```

```typescript
// src/index.ts
export default {
  async scheduled(event: ScheduledEvent, env: Bindings) {
    await checkPendingBatchJobs(env);
  },
  async fetch(request: Request, env: Bindings) {
    return app.fetch(request, env);
  }
};

async function checkPendingBatchJobs(env: Bindings) {
  const storage = new Storage(env.DB);
  const pendingJobs = await storage.getPendingBatchJobs();

  for (const job of pendingJobs) {
    const status = await checkProviderBatchStatus(job, env);

    if (status === "completed") {
      const results = await fetchBatchResults(job, env);
      await storage.completeBatchJob(job.id, results);

      if (job.notification_email) {
        await sendEmail(job.notification_email, results, env);
      }
    } else if (status === "failed") {
      await storage.failBatchJob(job.id, status.error);
    }
  }
}
```

### Cloudflare Queues Configuration

```toml
# wrangler.toml
[[queues.producers]]
queue = "batch-check"
binding = "BATCH_QUEUE"

[[queues.consumers]]
queue = "batch-check"
max_batch_size = 10
max_retries = 3
```

```typescript
// Queue consumer
export default {
  async queue(batch: MessageBatch, env: Bindings) {
    for (const message of batch.messages) {
      const { jobId, attempt } = message.body;
      const job = await storage.getBatchJob(jobId);

      if (!job || job.status !== "processing") continue;

      const status = await checkProviderStatus(job, env);

      if (status === "completed") {
        await completeJob(job, env);
      } else if (status === "failed") {
        await failJob(job, env);
      } else {
        // Reschedule with backoff
        const delay = getBackoffDelay(attempt);
        await env.BATCH_QUEUE.send(
          { jobId, attempt: attempt + 1 },
          { delaySeconds: delay }
        );
      }
    }
  }
};
```

### Batch Jobs Database Schema

```sql
CREATE TABLE batch_jobs (
  id TEXT PRIMARY KEY,
  conversation_id TEXT NOT NULL,
  user_query TEXT NOT NULL,
  flags TEXT NOT NULL,  -- JSON
  notification_email TEXT,

  status TEXT NOT NULL DEFAULT 'pending',
  -- pending, submitting, processing, completed, failed

  created_at TEXT NOT NULL,
  submitted_at TEXT,
  completed_at TEXT,
  last_checked_at TEXT,
  next_check_at TEXT,

  -- Provider batch IDs
  openai_batch_id TEXT,
  anthropic_batch_id TEXT,
  google_batch_id TEXT,

  -- Results (JSON blobs)
  stage1_results TEXT,
  stage2_results TEXT,
  stage3_result TEXT,

  error TEXT,
  check_count INTEGER DEFAULT 0,

  FOREIGN KEY (conversation_id) REFERENCES conversations(id)
);

CREATE INDEX idx_batch_jobs_status ON batch_jobs(status);
CREATE INDEX idx_batch_jobs_next_check ON batch_jobs(next_check_at);
```

### Email Notification Options

| Service | Free Tier | Cost After |
|---------|-----------|------------|
| **Resend** | 3,000/month | $0.001/email |
| **Mailgun** | 1,000/month | $0.80/1000 |
| **SendGrid** | 100/day | $0.001/email |

**Recommendation: Resend** - generous free tier, simple API:

```typescript
async function sendCompletionEmail(
  email: string,
  job: BatchJob,
  env: Bindings
) {
  await fetch("https://api.resend.com/emails", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${env.RESEND_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      from: "council@yourdomain.com",
      to: email,
      subject: `Council Analysis Complete: ${job.user_query.slice(0, 50)}...`,
      html: `
        <h2>Your LLM Council analysis is ready</h2>
        <p><strong>Query:</strong> ${job.user_query}</p>
        <p><a href="https://yourapp.com/conversations/${job.conversation_id}">
          View Results
        </a></p>
      `,
    }),
  });
}
```

### Batch Handling Cost Summary

| Component | Monthly Cost |
|-----------|--------------|
| Cloudflare Workers (paid) | $5 base |
| Cron Triggers | Free |
| Queues (< 1M messages) | Free |
| D1 Storage | ~$0.50 |
| Resend emails (< 3K) | Free |
| **Total Infrastructure** | **~$5.50/month** |

---

## 12. Repository Structure Recommendation

### Comparison of Approaches

| Approach | Complexity | Type Sharing | CI/CD | Best For |
|----------|------------|--------------|-------|----------|
| **Monorepo** (recommended) | Medium | Native | Single pipeline | Teams, long-term projects |
| **Two repos** | Low | NPM package | Two pipelines | Very separate teams |
| **Single repo, no workspaces** | Lowest | Direct imports | Single pipeline | Solo dev, small projects |

### Recommended: Bun Workspaces Monorepo

```
llm-council/
├── package.json              # Workspace root
├── bun.lockb
├── turbo.json                # Build orchestration
├── tsconfig.base.json        # Shared TS config
│
├── apps/
│   ├── api/                  # Hono backend → Cloudflare Workers
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── routes/
│   │   │   │   ├── conversations.ts
│   │   │   │   └── batch.ts
│   │   │   └── services/
│   │   │       ├── council.ts
│   │   │       ├── openrouter.ts
│   │   │       ├── storage.ts
│   │   │       └── batch.ts
│   │   ├── wrangler.toml
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── web/                  # Preact frontend → Cloudflare Pages
│       ├── src/
│       │   ├── app.tsx
│       │   ├── components/
│       │   │   ├── ChatInterface.tsx
│       │   │   ├── Sidebar.tsx
│       │   │   ├── Stage1.tsx
│       │   │   ├── Stage2.tsx
│       │   │   ├── Stage3.tsx
│       │   │   └── SettingsPanel.tsx
│       │   └── api.ts
│       ├── index.html
│       ├── vite.config.ts
│       ├── package.json
│       └── tsconfig.json
│
└── packages/
    └── shared/               # Shared types
        ├── src/
        │   ├── types.ts
        │   └── index.ts
        ├── package.json
        └── tsconfig.json
```

### Root Configuration Files

**package.json:**
```json
{
  "name": "llm-council",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "deploy": "turbo run deploy",
    "dev:api": "bun run --filter @llm-council/api dev",
    "dev:web": "bun run --filter @llm-council/web dev"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "typescript": "^5.4.0"
  }
}
```

**turbo.json:**
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "deploy": {
      "dependsOn": ["build"]
    },
    "typecheck": {
      "dependsOn": ["^build"]
    }
  }
}
```

**tsconfig.base.json:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "composite": true
  }
}
```

### Package Configurations

**packages/shared/package.json:**
```json
{
  "name": "@llm-council/shared",
  "version": "0.0.1",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch"
  },
  "devDependencies": {
    "typescript": "^5.4.0"
  }
}
```

**apps/api/package.json:**
```json
{
  "name": "@llm-council/api",
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "wrangler dev",
    "build": "wrangler deploy --dry-run --outdir=dist",
    "deploy": "wrangler deploy",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "hono": "^4.0.0",
    "@llm-council/shared": "workspace:*"
  },
  "devDependencies": {
    "wrangler": "^3.0.0",
    "@cloudflare/workers-types": "^4.0.0",
    "typescript": "^5.4.0"
  }
}
```

**apps/web/package.json:**
```json
{
  "name": "@llm-council/web",
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "deploy": "wrangler pages deploy dist --project-name=llm-council",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "preact": "^10.20.0",
    "@llm-council/shared": "workspace:*"
  },
  "devDependencies": {
    "@preact/preset-vite": "^2.8.0",
    "vite": "^5.0.0",
    "typescript": "^5.4.0"
  }
}
```

### Why Monorepo Over Two Repos

**Problems with two separate repos:**
1. **Type sharing** - Need to publish `@llm-council/shared` to NPM or duplicate types
2. **Version drift** - API changes break frontend, hard to track
3. **Two CI/CD pipelines** - More maintenance
4. **Separate installs** - Different lockfiles can cause issues

**When two repos makes sense:**
- Different teams own each part
- Very different release cycles
- Backend serves multiple frontends

**For solo/small team: Monorepo is clearly better.**

### Migration File Mapping

| Original Python | New TypeScript |
|-----------------|----------------|
| `backend/config.py` | `apps/api/src/config.ts` |
| `backend/openrouter.py` | `apps/api/src/services/openrouter.ts` |
| `backend/council.py` | `apps/api/src/services/council.ts` |
| `backend/storage.py` | `apps/api/src/services/storage.ts` |
| `backend/main.py` | `apps/api/src/index.ts` + `routes/*.ts` |
| `frontend/src/App.jsx` | `apps/web/src/app.tsx` |
| `frontend/src/api.js` | `apps/web/src/api.ts` |
| `frontend/src/components/*.jsx` | `apps/web/src/components/*.tsx` |
| `frontend/src/*.css` | `apps/web/src/*.css` (copy directly) |

### React to Preact Changes

| React | Preact |
|-------|--------|
| `import { useState } from 'react'` | `import { useState } from 'preact/hooks'` |
| `className="..."` | `class="..."` |
| `ReactMarkdown` | Use `marked` + `dangerouslySetInnerHTML` |
| `React.FC<Props>` | `FunctionComponent<Props>` |

### Quick Start Script

Save as `setup.sh`:

```bash
#!/bin/bash
set -e

# Root
bun init -y
cat > package.json << 'EOF'
{
  "name": "llm-council",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "typescript": "^5.4.0"
  }
}
EOF

# Create structure
mkdir -p apps/api/src/{routes,services} apps/web/src/components packages/shared/src

# Shared types
cat > packages/shared/package.json << 'EOF'
{
  "name": "@llm-council/shared",
  "version": "0.0.1",
  "type": "module",
  "main": "./src/index.ts",
  "types": "./src/index.ts"
}
EOF

cat > packages/shared/src/index.ts << 'EOF'
export * from './types';
EOF

cat > packages/shared/src/types.ts << 'EOF'
export interface Conversation {
  id: string;
  created_at: string;
  title: string;
  messages: Message[];
}

export interface Message {
  role: "user" | "assistant";
  content?: string;
  stage1?: Stage1Result[];
  stage2?: Stage2Result[];
  stage3?: Stage3Result;
}

export interface Stage1Result {
  model: string;
  response: string;
}

export interface Stage2Result {
  model: string;
  ranking: string;
  parsed_ranking: string[];
}

export interface Stage3Result {
  model: string;
  response: string;
}

export interface FeatureFlags {
  processing_mode: "realtime" | "batch";
  model_tier: "premium" | "budget" | "custom";
  council_size: number;
  skip_stage2: boolean;
}
EOF

# API
cd apps/api
bun init -y
bun add hono
bun add -d wrangler @cloudflare/workers-types typescript

cat > wrangler.toml << 'EOF'
name = "llm-council-api"
main = "src/index.ts"
compatibility_date = "2024-01-01"
EOF

cat > src/index.ts << 'EOF'
import { Hono } from "hono";
import { cors } from "hono/cors";

const app = new Hono();

app.use("/*", cors());
app.get("/", (c) => c.json({ status: "ok" }));
app.get("/api/health", (c) => c.json({ healthy: true }));

export default app;
EOF

# Web
cd ../web
bun create vite . --template preact-ts --yes
bun add @llm-council/shared@workspace:*

cd ../..
bun install

echo "Done! Run 'bun run dev' to start both servers."
```

### Development Workflow

```bash
# Terminal 1: API dev server (port 8787)
cd apps/api
bun run dev

# Terminal 2: Web dev server (port 5173)
cd apps/web
bun run dev

# Or from root with turbo:
bun run dev  # Runs both in parallel
```

### Deployment Commands

```bash
# Setup D1 database
cd apps/api
wrangler d1 create llm-council
# Copy database_id to wrangler.toml

# Apply schema
wrangler d1 execute llm-council --file=schema.sql

# Set secrets
wrangler secret put OPENROUTER_API_KEY

# Deploy API
bun run deploy

# Deploy frontend
cd ../web
bun run build
bun run deploy
```

### GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1

      - name: Install dependencies
        run: bun install

      - name: Build
        run: bun run build

      - name: Deploy API
        run: bunx wrangler deploy
        working-directory: apps/api
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}

      - name: Deploy Frontend
        run: bunx wrangler pages deploy dist --project-name=llm-council
        working-directory: apps/web
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

---

## Sources

- [Cloudflare Workers Pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Cloudflare Workers 5-minute CPU Limits](https://developers.cloudflare.com/changelog/2025-03-25-higher-cpu-limits/)
- [Cloudflare D1 Pricing](https://developers.cloudflare.com/d1/platform/pricing/)
- [Cloudflare Queues](https://developers.cloudflare.com/queues/)
- [Cloudflare Cron Triggers](https://developers.cloudflare.com/workers/configuration/cron-triggers/)
- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Google Gemini Batch API](https://cloud.google.com/vertex-ai/docs/generative-ai/batch-prediction)
- [Hono Documentation](https://hono.dev/docs/)
- [Preact Documentation](https://preactjs.com/guide/v10/getting-started)
- [Bun Workspaces](https://bun.sh/docs/install/workspaces)
- [Turborepo](https://turbo.build/repo/docs)
- [Resend Email API](https://resend.com/docs)
- [OpenRouter Pricing](https://openrouter.ai/pricing)

---

## 13. Authentication: WorkOS AuthKit

### Why WorkOS

| Factor | WorkOS | Clerk | Supabase |
|--------|--------|-------|----------|
| **Free tier** | **1M MAU** | 10K MAU | 50K MAU |
| **Setup time** | ~2 hours | ~1 hour | ~1 hour |
| **Production restrictions** | None | Yes | None |
| **Enterprise SSO** | Included | Paid | Paid |
| **Magic links** | Yes | Yes | Yes |
| **User allowlists** | Yes | Yes | Yes |

### Setup Time Breakdown

| Step | Time |
|------|------|
| Create WorkOS account | 5 min |
| Dashboard configuration | 15 min |
| Frontend auth code | 30 min |
| Backend auth routes | 45 min |
| Testing | 15 min |
| **Total** | **~2 hours** |

### WorkOS Dashboard Configuration

1. **User Management** → Enable AuthKit
2. **Authentication Methods**:
   - ✅ Email + Password
   - ✅ Magic Link
   - ✅ Google OAuth
   - ✅ GitHub OAuth
3. **Redirects**:
   ```
   http://localhost:5173/auth/callback     (dev)
   https://yourapp.com/auth/callback       (prod)
   ```
4. Copy credentials:
   - Client ID: `client_...`
   - API Key: `sk_...`

### Access Control Options

#### Option 1: Domain Restriction

Only allow specific email domains:

**Dashboard → User Management → Domain Restrictions**
```
Allowed domains:
  yourcompany.com
  partner.com
```

#### Option 2: Invite-Only Mode

**Dashboard → User Management → Settings**
- ❌ Disable "Allow self-service sign-up"
- ✅ Enable "Invite-only"

Invite via API:
```bash
curl -X POST https://api.workos.com/user_management/invitations \
  -H "Authorization: Bearer sk_..." \
  -d '{"email": "user@example.com"}'
```

#### Option 3: D1 Allowlist

```sql
CREATE TABLE allowed_users (
  email TEXT PRIMARY KEY,
  added_at TEXT DEFAULT CURRENT_TIMESTAMP,
  added_by TEXT
);

INSERT INTO allowed_users (email) VALUES
  ('you@example.com'),
  ('teammate@example.com');
```

```typescript
// Check allowlist after WorkOS auth
authRouter.post('/callback', async (c) => {
  // ... WorkOS token exchange ...

  const email = data.user.email;

  const allowed = await c.env.DB
    .prepare('SELECT 1 FROM allowed_users WHERE email = ?')
    .bind(email)
    .first();

  if (!allowed) {
    return c.json({ error: 'Access denied' }, 403);
  }

  // Continue with login...
});
```

### Frontend Implementation

```typescript
// apps/web/src/lib/auth.ts
const WORKOS_CLIENT_ID = import.meta.env.VITE_WORKOS_CLIENT_ID;
const REDIRECT_URI = import.meta.env.VITE_WORKOS_REDIRECT_URI;

export function getAuthUrl(provider?: 'GoogleOAuth' | 'GitHubOAuth') {
  const params = new URLSearchParams({
    client_id: WORKOS_CLIENT_ID,
    redirect_uri: REDIRECT_URI,
    response_type: 'code',
    ...(provider && { provider }),
  });
  return `https://api.workos.com/user_management/authorize?${params}`;
}

export const signIn = () => window.location.href = getAuthUrl();
export const signInWithGoogle = () => window.location.href = getAuthUrl('GoogleOAuth');
export const signInWithGithub = () => window.location.href = getAuthUrl('GitHubOAuth');
```

```typescript
// apps/web/src/hooks/useAuth.ts
import { useState, useEffect, useCallback } from 'preact/hooks';

interface User {
  id: string;
  email: string;
  firstName?: string;
  lastName?: string;
  profilePictureUrl?: string;
}

export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [token, setToken] = useState<string | null>(null);

  useEffect(() => {
    const stored = localStorage.getItem('auth');
    if (stored) {
      const { user, token, expiresAt } = JSON.parse(stored);
      if (expiresAt > Date.now()) {
        setUser(user);
        setToken(token);
      } else {
        localStorage.removeItem('auth');
      }
    }
    setLoading(false);
  }, []);

  const handleCallback = useCallback(async (code: string) => {
    const res = await fetch(`${API_URL}/api/auth/callback`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ code }),
    });
    if (!res.ok) throw new Error('Auth failed');
    const data = await res.json();

    const auth = {
      user: data.user,
      token: data.accessToken,
      expiresAt: Date.now() + (data.expiresIn * 1000),
    };
    localStorage.setItem('auth', JSON.stringify(auth));
    setUser(data.user);
    setToken(data.accessToken);
    return data.user;
  }, []);

  const signOut = useCallback(() => {
    localStorage.removeItem('auth');
    setUser(null);
    setToken(null);
  }, []);

  const authFetch = useCallback((url: string, options: RequestInit = {}) => {
    return fetch(url, {
      ...options,
      headers: { ...options.headers, Authorization: `Bearer ${token}` },
    });
  }, [token]);

  return { user, token, loading, isAuthenticated: !!user, handleCallback, signOut, authFetch };
}
```

### Backend Implementation

```typescript
// apps/api/src/routes/auth.ts
import { Hono } from 'hono';

export const authRouter = new Hono<{ Bindings: Bindings }>();

authRouter.post('/callback', async (c) => {
  const { code } = await c.req.json();

  const res = await fetch('https://api.workos.com/user_management/authenticate', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      client_id: c.env.WORKOS_CLIENT_ID,
      client_secret: c.env.WORKOS_API_KEY,
      grant_type: 'authorization_code',
      code,
    }),
  });

  if (!res.ok) return c.json({ error: 'Authentication failed' }, 401);

  const data = await res.json();

  // Upsert user in D1
  await c.env.DB.prepare(`
    INSERT INTO users (id, email, first_name, last_name, profile_picture_url)
    VALUES (?, ?, ?, ?, ?)
    ON CONFLICT(id) DO UPDATE SET
      email = excluded.email,
      first_name = excluded.first_name,
      last_name = excluded.last_name,
      profile_picture_url = excluded.profile_picture_url
  `).bind(
    data.user.id,
    data.user.email,
    data.user.first_name,
    data.user.last_name,
    data.user.profile_picture_url
  ).run();

  return c.json({
    user: {
      id: data.user.id,
      email: data.user.email,
      firstName: data.user.first_name,
      lastName: data.user.last_name,
      profilePictureUrl: data.user.profile_picture_url,
    },
    accessToken: data.access_token,
    refreshToken: data.refresh_token,
    expiresIn: data.expires_in,
  });
});
```

```typescript
// apps/api/src/middleware/auth.ts
import { createMiddleware } from 'hono/factory';
import { jwtVerify, createRemoteJWKSet } from 'jose';

const JWKS = createRemoteJWKSet(new URL('https://api.workos.com/sso/jwks'));

export const requireAuth = createMiddleware(async (c, next) => {
  const auth = c.req.header('Authorization');
  if (!auth?.startsWith('Bearer ')) {
    return c.json({ error: 'Unauthorized' }, 401);
  }

  try {
    const { payload } = await jwtVerify(auth.slice(7), JWKS);
    c.set('userId', payload.sub);
    await next();
  } catch {
    return c.json({ error: 'Invalid token' }, 401);
  }
});
```

### Custom SMTP (Use Postmark for Magic Links)

Route WorkOS emails through your Postmark account:

**Dashboard → Email → Custom SMTP**
```
SMTP Host: smtp.postmarkapp.com
Port: 587
Username: your-postmark-api-token
Password: your-postmark-api-token
From: auth@yourdomain.com
```

---

## 14. Final Cost Summary

### Infrastructure (Fixed Monthly)

| Component | Cost | Notes |
|-----------|------|-------|
| Cloudflare Workers | $5.00 | Paid tier for 5-min CPU |
| Cloudflare D1 | ~$0.50 | Database |
| Cloudflare Pages | Free | Frontend hosting |
| Cloudflare Queues/Cron | Free | Batch job polling |
| WorkOS Auth | Free | 1M MAU |
| Postmark (shared 5-way) | $3.00 | Email notifications + magic links |
| Domain | ~$1.00 | |
| **Total Infrastructure** | **$9.50/mo** | |

### LLM API Costs (Per Query)

| Mode | Cost/Query | Description |
|------|------------|-------------|
| Premium Realtime | $0.175 | GPT-5.1, Claude Sonnet, Gemini Pro, Grok |
| **Premium Batch** | **$0.088** | 50% off via direct provider APIs |
| Budget Realtime | $0.035 | DeepSeek, Haiku, Gemini Flash, Grok-fast |
| **Budget Batch** | **$0.018** | 50% off with cheap models |

### Premium + Batch Cost Breakdown ($0.088/query)

| Stage | Realtime Cost | Batch (50% off) |
|-------|---------------|-----------------|
| Stage 1 (4 models) | $0.074 | $0.037 |
| Stage 2 (4 models) | $0.060 | $0.030 |
| Stage 3 (1 model) | $0.040 | $0.020 |
| **Total** | **$0.175** | **$0.088** |

### Total Monthly Costs by Usage (All Four Options)

| Queries/Day | Queries/Month | Budget+Batch | Premium+Batch | Budget RT | Premium RT |
|-------------|---------------|--------------|---------------|-----------|------------|
| 10 | 300 | **$15** | $36 | $20 | $62 |
| 50 | 1,500 | **$37** | $142 | $62 | $272 |
| 100 | 3,000 | **$63** | $274 | $115 | $535 |
| 200 | 6,000 | **$118** | $538 | $220 | $1,060 |
| 500 | 15,000 | **$279** | $1,330 | $535 | $2,635 |
| 1,000 | 30,000 | **$549** | $2,650 | $1,060 | $5,260 |

### Cost Comparison Summary

| Mode | vs Premium RT Savings |
|------|----------------------|
| Premium Realtime | baseline |
| Premium Batch | **50% cheaper** |
| Budget Realtime | **80% cheaper** |
| Budget Batch | **90% cheaper** |

### Cost Formula

```
Monthly Cost = $9.50 + (queries × LLM rate)

LLM Rates:
  • Budget + Batch:    $0.018/query  (cheapest)
  • Budget Realtime:   $0.035/query
  • Premium + Batch:   $0.088/query
  • Premium Realtime:  $0.175/query  (most expensive)
```

### Quick Reference Card

```
┌────────────────────────────────────────────────────────────┐
│              LLM COUNCIL - MONTHLY COST CALCULATOR         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Fixed Infrastructure:            $9.50/month              │
│    • Cloudflare (Workers + D1)    $5.50                   │
│    • Postmark (shared)            $3.00                    │
│    • Domain                       $1.00                    │
│    • WorkOS, Pages, Queues        Free                     │
│                                                            │
│  LLM API Rates:                                            │
│    • Budget + Batch:     $0.018/query  (recommended)      │
│    • Budget Realtime:    $0.035/query                     │
│    • Premium + Batch:    $0.088/query                     │
│    • Premium Realtime:   $0.175/query                     │
│                                                            │
│  Examples (Budget + Batch):                                │
│    10/day   = $9.50 + $5.40    = $15/month                │
│    50/day   = $9.50 + $27      = $37/month                │
│    200/day  = $9.50 + $108     = $118/month               │
│    1K/day   = $9.50 + $540     = $549/month               │
│                                                            │
│  Examples (Premium + Batch):                               │
│    10/day   = $9.50 + $26      = $36/month                │
│    50/day   = $9.50 + $132     = $142/month               │
│    200/day  = $9.50 + $528     = $538/month               │
│    1K/day   = $9.50 + $2,640   = $2,650/month             │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Postmark Integration for Batch Notifications

```typescript
// apps/api/src/services/email.ts
export async function sendBatchCompleteEmail(
  to: string,
  job: BatchJob,
  env: Bindings
) {
  await fetch('https://api.postmarkapp.com/email', {
    method: 'POST',
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
      'X-Postmark-Server-Token': env.POSTMARK_API_TOKEN,
    },
    body: JSON.stringify({
      From: 'council@yourdomain.com',
      To: to,
      Subject: `Council Analysis Ready: ${job.query.slice(0, 40)}...`,
      HtmlBody: `
        <h2>Your analysis is complete</h2>
        <p><strong>Query:</strong> ${job.query}</p>
        <p><a href="https://yourapp.com/c/${job.conversationId}">View Results</a></p>
      `,
      MessageStream: 'outbound',
    }),
  });
}
```

```bash
# Add Postmark secret
wrangler secret put POSTMARK_API_TOKEN
```

---

## 15. Complete Tech Stack Summary

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER                                        │
│                          │                                          │
│                          ▼                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  CLOUDFLARE EDGE                             │   │
│  │  ┌─────────────┐                      ┌─────────────┐       │   │
│  │  │   PAGES     │    ┌─────────────┐   │   WORKERS   │       │   │
│  │  │  (Preact)   │───▶│   WorkOS    │   │   (Hono)    │       │   │
│  │  │             │    │   AuthKit   │   │             │       │   │
│  │  └─────────────┘    └──────┬──────┘   └──────┬──────┘       │   │
│  │                            │                  │              │   │
│  │                            ▼                  ▼              │   │
│  │                     ┌─────────────┐    ┌─────────────┐      │   │
│  │                     │     D1      │    │  QUEUES     │      │   │
│  │                     │  (SQLite)   │    │  + CRON     │      │   │
│  │                     └─────────────┘    └─────────────┘      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                │                                    │
│              ┌─────────────────┼─────────────────┐                 │
│              ▼                 ▼                 ▼                 │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐      │
│  │    OpenRouter   │ │  Direct APIs    │ │    Postmark     │      │
│  │   (Realtime)    │ │ (Batch 50% off) │ │    (Email)      │      │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Tech Stack Components

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Frontend** | Preact + Vite | UI (3KB bundle) |
| **Hosting** | Cloudflare Pages | Static hosting (free) |
| **Backend** | Hono on Workers | API (edge, $5/mo) |
| **Database** | Cloudflare D1 | SQLite (~$0.50/mo) |
| **Auth** | WorkOS AuthKit | Authentication (free 1M MAU) |
| **Queue** | Cloudflare Queues | Batch job polling (free) |
| **Scheduler** | Cloudflare Cron | Backup polling (free) |
| **Email** | Postmark | Notifications ($15/mo existing) |
| **LLM (RT)** | OpenRouter | Unified gateway |
| **LLM (Batch)** | Direct APIs | 50% discount |

### Development Timeline

| Phase | Tasks | Time |
|-------|-------|------|
| **Week 1** | Monorepo setup, backend port, D1 schema | 5 days |
| **Week 2** | Batch processing, WorkOS auth | 5 days |
| **Week 3** | Frontend port, settings UI, batch UI | 5 days |
| **Week 4** | Testing, deployment, polish | 5 days |
| **Total** | | **~4 weeks** |

### Files to Create

```
llm-council-v2/
├── package.json
├── turbo.json
├── tsconfig.base.json
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── routes/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── conversations.ts
│   │   │   │   └── batch.ts
│   │   │   ├── services/
│   │   │   │   ├── council.ts
│   │   │   │   ├── openrouter.ts
│   │   │   │   ├── storage.ts
│   │   │   │   ├── batch.ts
│   │   │   │   └── email.ts
│   │   │   └── middleware/
│   │   │       └── auth.ts
│   │   ├── wrangler.toml
│   │   └── package.json
│   └── web/
│       ├── src/
│       │   ├── app.tsx
│       │   ├── main.tsx
│       │   ├── api.ts
│       │   ├── lib/
│       │   │   └── auth.ts
│       │   ├── hooks/
│       │   │   ├── useAuth.ts
│       │   │   └── useConversation.ts
│       │   └── components/
│       │       ├── ChatInterface.tsx
│       │       ├── Sidebar.tsx
│       │       ├── SettingsPanel.tsx
│       │       ├── BatchJobsPanel.tsx
│       │       ├── Stage1.tsx
│       │       ├── Stage2.tsx
│       │       └── Stage3.tsx
│       ├── index.html
│       ├── vite.config.ts
│       └── package.json
└── packages/
    └── shared/
        ├── src/
        │   ├── types.ts
        │   └── index.ts
        └── package.json
```

---

## Sources

- [Cloudflare Workers Pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Cloudflare Workers 5-minute CPU Limits](https://developers.cloudflare.com/changelog/2025-03-25-higher-cpu-limits/)
- [Cloudflare D1 Pricing](https://developers.cloudflare.com/d1/platform/pricing/)
- [Cloudflare Queues](https://developers.cloudflare.com/queues/)
- [Cloudflare Cron Triggers](https://developers.cloudflare.com/workers/configuration/cron-triggers/)
- [WorkOS AuthKit](https://workos.com/docs/user-management)
- [WorkOS Pricing](https://workos.com/pricing) (1M MAU free)
- [Postmark API](https://postmarkapp.com/developer)
- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Google Gemini Batch API](https://cloud.google.com/vertex-ai/docs/generative-ai/batch-prediction)
- [Hono Documentation](https://hono.dev/docs/)
- [Preact Documentation](https://preactjs.com/guide/v10/getting-started)
- [Bun Workspaces](https://bun.sh/docs/install/workspaces)
- [Turborepo](https://turbo.build/repo/docs)
- [OpenRouter Pricing](https://openrouter.ai/pricing)

---

## 16. Analytics & Historical Data Storage

### What Gets Saved

| Data | Stored | Purpose |
|------|--------|---------|
| User queries | ✅ Yes | Historical reference |
| All Stage 1 responses | ✅ Yes | Full model outputs |
| All Stage 2 rankings | ✅ Yes | Peer evaluations |
| Stage 3 synthesis | ✅ Yes | Final answers |
| Token usage | ✅ Yes | Cost tracking |
| Response times | ✅ Yes | Performance optimization |
| Model win rates | ✅ Yes | Model selection tuning |
| Query hashes | ✅ Yes | Duplicate/cache detection |

### Analytics Schema

```sql
-- Query analytics (one row per query)
CREATE TABLE query_analytics (
  id INTEGER PRIMARY KEY,
  message_id INTEGER NOT NULL,
  user_id TEXT NOT NULL,

  -- Query metadata
  query_length INTEGER,
  query_hash TEXT,                -- for duplicate detection

  -- Feature flags used
  processing_mode TEXT,           -- 'realtime' or 'batch'
  model_tier TEXT,                -- 'premium', 'budget', 'custom'
  council_size INTEGER,
  skip_stage2 BOOLEAN,

  -- Models used
  council_models TEXT,            -- JSON array
  chairman_model TEXT,

  -- Performance metrics
  stage1_duration_ms INTEGER,
  stage2_duration_ms INTEGER,
  stage3_duration_ms INTEGER,
  total_duration_ms INTEGER,

  -- Token usage
  stage1_input_tokens INTEGER,
  stage1_output_tokens INTEGER,
  stage2_input_tokens INTEGER,
  stage2_output_tokens INTEGER,
  stage3_input_tokens INTEGER,
  stage3_output_tokens INTEGER,
  total_tokens INTEGER,

  -- Cost tracking
  estimated_cost_usd REAL,

  -- Model performance
  models_succeeded INTEGER,
  models_failed INTEGER,
  winning_model TEXT,
  ranking_agreement REAL,         -- 0-1 consensus score

  created_at TEXT,

  FOREIGN KEY (message_id) REFERENCES messages(id)
);

CREATE INDEX idx_analytics_user ON query_analytics(user_id);
CREATE INDEX idx_analytics_date ON query_analytics(created_at);
CREATE INDEX idx_analytics_tier ON query_analytics(model_tier);
CREATE INDEX idx_analytics_hash ON query_analytics(query_hash);

-- Per-model performance tracking
CREATE TABLE model_performance (
  id INTEGER PRIMARY KEY,
  query_analytics_id INTEGER,
  model TEXT NOT NULL,

  -- Response metrics
  response_time_ms INTEGER,
  output_tokens INTEGER,

  -- Quality signals
  was_ranked_first INTEGER,       -- 1 if ranked #1
  average_rank REAL,

  -- Errors
  failed BOOLEAN DEFAULT FALSE,
  error_type TEXT,                -- 'timeout', 'rate_limit', 'api_error'

  created_at TEXT,

  FOREIGN KEY (query_analytics_id) REFERENCES query_analytics(id)
);

CREATE INDEX idx_model_perf_model ON model_performance(model);
CREATE INDEX idx_model_perf_date ON model_performance(created_at);

-- Daily aggregates (for dashboards)
CREATE TABLE daily_stats (
  date TEXT,
  user_id TEXT,

  total_queries INTEGER,
  realtime_queries INTEGER,
  batch_queries INTEGER,

  total_tokens INTEGER,
  total_cost_usd REAL,

  avg_duration_ms REAL,

  model_rankings TEXT,            -- JSON: {"gpt-5.1": 0.35, ...}

  PRIMARY KEY (date, user_id)
);
```

### Analytics Tracking Implementation

```typescript
// apps/api/src/services/analytics.ts

interface QueryAnalytics {
  messageId: number;
  userId: string;
  queryLength: number;
  queryHash: string;
  processingMode: string;
  modelTier: string;
  councilSize: number;
  skipStage2: boolean;
  councilModels: string[];
  chairmanModel: string;
  stage1DurationMs: number;
  stage2DurationMs: number;
  stage3DurationMs: number;
  totalDurationMs: number;
  stage1InputTokens: number;
  stage1OutputTokens: number;
  stage2InputTokens: number;
  stage2OutputTokens: number;
  stage3InputTokens: number;
  stage3OutputTokens: number;
  totalTokens: number;
  estimatedCostUsd: number;
  modelsSucceeded: number;
  modelsFailed: number;
  winningModel: string | null;
  rankingAgreement: number;
}

export async function trackQueryAnalytics(
  db: D1Database,
  analytics: QueryAnalytics
): Promise<number> {
  const result = await db.prepare(`
    INSERT INTO query_analytics (
      message_id, user_id, query_length, query_hash,
      processing_mode, model_tier, council_size, skip_stage2,
      council_models, chairman_model,
      stage1_duration_ms, stage2_duration_ms, stage3_duration_ms, total_duration_ms,
      stage1_input_tokens, stage1_output_tokens,
      stage2_input_tokens, stage2_output_tokens,
      stage3_input_tokens, stage3_output_tokens,
      total_tokens, estimated_cost_usd,
      models_succeeded, models_failed,
      winning_model, ranking_agreement,
      created_at
    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    RETURNING id
  `).bind(
    analytics.messageId,
    analytics.userId,
    analytics.queryLength,
    analytics.queryHash,
    analytics.processingMode,
    analytics.modelTier,
    analytics.councilSize,
    analytics.skipStage2 ? 1 : 0,
    JSON.stringify(analytics.councilModels),
    analytics.chairmanModel,
    analytics.stage1DurationMs,
    analytics.stage2DurationMs,
    analytics.stage3DurationMs,
    analytics.totalDurationMs,
    analytics.stage1InputTokens,
    analytics.stage1OutputTokens,
    analytics.stage2InputTokens,
    analytics.stage2OutputTokens,
    analytics.stage3InputTokens,
    analytics.stage3OutputTokens,
    analytics.totalTokens,
    analytics.estimatedCostUsd,
    analytics.modelsSucceeded,
    analytics.modelsFailed,
    analytics.winningModel,
    analytics.rankingAgreement,
    new Date().toISOString()
  ).first<{ id: number }>();

  return result?.id ?? 0;
}

export async function trackModelPerformance(
  db: D1Database,
  analyticsId: number,
  model: string,
  metrics: {
    responseTimeMs: number;
    outputTokens: number;
    wasRankedFirst: boolean;
    averageRank: number;
    failed: boolean;
    errorType?: string;
  }
) {
  await db.prepare(`
    INSERT INTO model_performance (
      query_analytics_id, model,
      response_time_ms, output_tokens,
      was_ranked_first, average_rank,
      failed, error_type,
      created_at
    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
  `).bind(
    analyticsId,
    model,
    metrics.responseTimeMs,
    metrics.outputTokens,
    metrics.wasRankedFirst ? 1 : 0,
    metrics.averageRank,
    metrics.failed ? 1 : 0,
    metrics.errorType ?? null,
    new Date().toISOString()
  ).run();
}

// Hash query for duplicate detection
export async function hashQuery(query: string): Promise<string> {
  const normalized = query.toLowerCase().trim();
  const buffer = await crypto.subtle.digest(
    'SHA-256',
    new TextEncoder().encode(normalized)
  );
  return Array.from(new Uint8Array(buffer))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('')
    .slice(0, 16);
}

// Calculate ranking agreement (consensus score)
export function calculateRankingAgreement(stage2Results: Stage2Result[]): number {
  if (stage2Results.length < 2) return 1;

  let agreements = 0;
  let comparisons = 0;

  for (let i = 0; i < stage2Results.length; i++) {
    for (let j = i + 1; j < stage2Results.length; j++) {
      const r1 = stage2Results[i].parsed_ranking;
      const r2 = stage2Results[j].parsed_ranking;
      const matches = r1.filter((item, idx) => r2[idx] === item).length;
      agreements += matches / r1.length;
      comparisons++;
    }
  }

  return comparisons > 0 ? agreements / comparisons : 1;
}
```

### Integration with Council Flow

```typescript
// apps/api/src/services/council.ts

export async function runCouncil(
  userQuery: string,
  flags: FeatureFlags,
  env: Bindings,
  userId: string,
  messageId: number
) {
  const startTime = Date.now();
  const { council, chairman } = resolveModels(flags);

  // Stage 1 with timing
  const stage1Start = Date.now();
  const stage1Results = await queryModelsParallel(council, messages, env);
  const stage1Duration = Date.now() - stage1Start;

  // Stage 2 with timing (if not skipped)
  let stage2Results: Stage2Result[] = [];
  let stage2Duration = 0;
  if (!flags.skip_stage2) {
    const stage2Start = Date.now();
    stage2Results = await collectRankings(userQuery, stage1Results, council, env);
    stage2Duration = Date.now() - stage2Start;
  }

  // Stage 3 with timing
  const stage3Start = Date.now();
  const stage3Result = await synthesize(userQuery, stage1Results, stage2Results, chairman, env);
  const stage3Duration = Date.now() - stage3Start;

  const totalDuration = Date.now() - startTime;

  // Calculate metrics
  const tokens = estimateTokens(userQuery, stage1Results, stage2Results, stage3Result);
  const cost = calculateCost(tokens, flags);
  const aggregateRankings = calculateAggregateRankings(stage2Results, labelToModel);
  const winningModel = aggregateRankings[0]?.model ?? null;
  const rankingAgreement = calculateRankingAgreement(stage2Results);

  // Track analytics
  const analyticsId = await trackQueryAnalytics(env.DB, {
    messageId,
    userId,
    queryLength: estimateTokenCount(userQuery),
    queryHash: await hashQuery(userQuery),
    processingMode: flags.processing_mode,
    modelTier: flags.model_tier,
    councilSize: flags.council_size,
    skipStage2: flags.skip_stage2,
    councilModels: council,
    chairmanModel: chairman,
    stage1DurationMs: stage1Duration,
    stage2DurationMs: stage2Duration,
    stage3DurationMs: stage3Duration,
    totalDurationMs: totalDuration,
    ...tokens,
    estimatedCostUsd: cost,
    modelsSucceeded: stage1Results.length,
    modelsFailed: council.length - stage1Results.length,
    winningModel,
    rankingAgreement,
  });

  // Track per-model performance
  for (const result of stage1Results) {
    const modelRanking = aggregateRankings.find(r => r.model === result.model);
    await trackModelPerformance(env.DB, analyticsId, result.model, {
      responseTimeMs: result.responseTime ?? 0,
      outputTokens: estimateTokenCount(result.response),
      wasRankedFirst: modelRanking?.average_rank === 1,
      averageRank: modelRanking?.average_rank ?? 0,
      failed: false,
    });
  }

  return { stage1Results, stage2Results, stage3Result, metadata };
}
```

### Useful Analytics Queries

```sql
-- Monthly cost summary
SELECT
  strftime('%Y-%m', created_at) as month,
  COUNT(*) as queries,
  SUM(estimated_cost_usd) as total_cost,
  AVG(estimated_cost_usd) as avg_cost
FROM query_analytics
GROUP BY month
ORDER BY month DESC;

-- Model win rates (which model ranks #1 most often)
SELECT
  model,
  COUNT(*) as times_first,
  ROUND(AVG(average_rank), 2) as avg_rank,
  ROUND(AVG(response_time_ms)) as avg_response_ms
FROM model_performance
WHERE failed = 0
GROUP BY model
ORDER BY times_first DESC;

-- Cost by tier
SELECT
  model_tier,
  COUNT(*) as queries,
  SUM(estimated_cost_usd) as total_cost,
  ROUND(AVG(estimated_cost_usd), 4) as avg_cost
FROM query_analytics
GROUP BY model_tier;

-- Duplicate queries (caching opportunities)
SELECT
  query_hash,
  COUNT(*) as occurrences,
  MIN(created_at) as first_seen
FROM query_analytics
GROUP BY query_hash
HAVING occurrences > 1
ORDER BY occurrences DESC
LIMIT 20;

-- Failed models (reliability issues)
SELECT
  model,
  error_type,
  COUNT(*) as failures
FROM model_performance
WHERE failed = 1
GROUP BY model, error_type
ORDER BY failures DESC;

-- Response time trends
SELECT
  date(created_at) as day,
  ROUND(AVG(total_duration_ms)) as avg_duration,
  ROUND(AVG(stage1_duration_ms)) as avg_stage1,
  ROUND(AVG(stage2_duration_ms)) as avg_stage2,
  ROUND(AVG(stage3_duration_ms)) as avg_stage3
FROM query_analytics
GROUP BY day
ORDER BY day DESC
LIMIT 30;

-- User usage summary
SELECT
  user_id,
  COUNT(*) as total_queries,
  SUM(estimated_cost_usd) as total_cost,
  MAX(created_at) as last_query
FROM query_analytics
GROUP BY user_id
ORDER BY total_queries DESC;

-- Ranking agreement analysis
SELECT
  CASE
    WHEN ranking_agreement >= 0.8 THEN 'High (0.8-1.0)'
    WHEN ranking_agreement >= 0.5 THEN 'Medium (0.5-0.8)'
    ELSE 'Low (0-0.5)'
  END as agreement_level,
  COUNT(*) as queries,
  ROUND(AVG(council_size), 1) as avg_council_size
FROM query_analytics
WHERE skip_stage2 = 0
GROUP BY agreement_level;
```

### Future Optimization Uses

| Data Collected | Optimization Opportunity |
|----------------|-------------------------|
| Query hashes with duplicates | Implement response caching |
| Model win rates | Adjust default council composition |
| Response times by model | Set appropriate timeouts |
| Token usage patterns | Optimize prompts, reduce costs |
| Cost by tier | Recommend tier based on usage |
| Ranking agreement | Use fewer models when agreement is high |
| Failed models | Remove unreliable models from council |
| User patterns | Personalized defaults |

### Storage Requirements

| Usage | Queries/Month | Analytics Data | Total D1 Size |
|-------|---------------|----------------|---------------|
| 10/day | 300 | ~3 MB | ~5 MB |
| 50/day | 1,500 | ~15 MB | ~25 MB |
| 200/day | 6,000 | ~60 MB | ~100 MB |
| 1K/day | 30,000 | ~300 MB | ~500 MB |

D1 free tier includes 5GB storage - sufficient for ~50K queries/month with full analytics.

### Data Retention Policy

```sql
-- Optional: Clean up old detailed analytics (keep aggregates)
DELETE FROM model_performance
WHERE created_at < date('now', '-90 days');

DELETE FROM query_analytics
WHERE created_at < date('now', '-90 days');

-- Keep daily_stats indefinitely for trends
```

---

*Last updated: January 2026*
