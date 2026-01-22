# Product Requirements Document (PRD)
# LLM Council v2.0 — Cloudflare Edge Refactor

**Version:** 1.0
**Date:** January 2026
**Author:** Generated from research findings
**Status:** Draft for Review

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Goals & Success Criteria](#2-goals--success-criteria)
3. [User Stories](#3-user-stories)
4. [Technical Architecture](#4-technical-architecture)
5. [Feature Specifications](#5-feature-specifications)
6. [Feature Flags System](#6-feature-flags-system)
7. [Authentication (WorkOS)](#7-authentication-workos)
8. [Database Schema (D1)](#8-database-schema-d1)
9. [API Specification](#9-api-specification)
10. [Testing Strategy](#10-testing-strategy)
11. [Deployment & CI/CD](#11-deployment--cicd)
12. [Cost Analysis](#12-cost-analysis)
13. [Security Requirements](#13-security-requirements)
14. [Migration Plan](#14-migration-plan)
15. [Risks & Mitigations](#15-risks--mitigations)
16. [RAG System (Future)](#16-rag-system-future)
17. [Appendix](#17-appendix)

---

## 1. Executive Summary

### Project Overview

Refactor LLM Council from Python/FastAPI + React to a modern, edge-deployed monorepo:

| Component | Current | Target |
|-----------|---------|--------|
| **Runtime** | Python + Node.js | Bun / Cloudflare Workers |
| **Backend** | FastAPI | Hono |
| **Frontend** | React 19 | Astro + Preact islands |
| **Auth** | None | WorkOS |
| **Storage** | JSON files | Cloudflare D1 (SQLite) |
| **Hosting** | Local/VPS | Cloudflare (Workers + Pages) |
| **Deploy** | Manual | GitHub Actions |
| **Config** | Hardcoded | Feature flags |

> **Note:** Cloudflare acquired Astro in January 2026, making it the recommended frontend framework for Cloudflare deployments.

### Key Drivers

1. **Cost efficiency**: $0-10/month infrastructure budget
2. **Robustness**: Comprehensive testing, error handling, monitoring
3. **Flexibility**: Feature flags for all major configurations
4. **Scalability**: Edge deployment, ready for team use (5-20 users)
5. **Maintainability**: TypeScript monorepo, shared types

### Out of Scope (v2.0)

- Multi-tenant SaaS / billing
- Mobile applications
- Offline support
- Real-time collaboration

---

## 2. Goals & Success Criteria

### Primary Goals

| Goal | Success Metric |
|------|----------------|
| **Zero-downtime deployment** | Deploy without user interruption |
| **Sub-$10/month infrastructure** | Cloudflare free/paid tier within budget |
| **<60s average response time** | End-to-end council query completion |
| **99% uptime** | Measured over 30-day rolling window |
| **100% feature flag coverage** | All configurable features toggleable |

### Quality Gates

| Gate | Requirement |
|------|-------------|
| **Unit test coverage** | ≥80% for core services |
| **Integration tests** | All API endpoints covered |
| **E2E tests** | Critical user flows automated |
| **Error handling** | All external calls have retry + fallback |
| **Type safety** | Zero `any` types in production code |

---

## 3. User Stories

### Authentication

| ID | Story | Priority |
|----|-------|----------|
| US-1 | As a user, I can sign up with email/password | P0 |
| US-2 | As a user, I can log in and maintain a session | P0 |
| US-3 | As a user, I can reset my password | P0 |
| US-4 | As an admin, I can invite team members | P1 |
| US-5 | As a user, I can log in via SSO (future) | P2 |

### Core Functionality

| ID | Story | Priority |
|----|-------|----------|
| US-10 | As a user, I can submit a query and receive a council response | P0 |
| US-11 | As a user, I can view all three stages of deliberation | P0 |
| US-12 | As a user, I can see aggregate rankings | P0 |
| US-13 | As a user, I can view my conversation history | P0 |
| US-14 | As a user, I can continue a previous conversation | P1 |

### Feature Flags

| ID | Story | Priority |
|----|-------|----------|
| US-20 | As an admin, I can switch between premium and budget models | P0 |
| US-21 | As an admin, I can enable/disable streaming responses | P0 |
| US-22 | As an admin, I can adjust council size (3-5 models) | P1 |
| US-23 | As an admin, I can change the chairman model | P1 |
| US-24 | As an admin, I can toggle anonymization in peer review | P1 |
| US-25 | As an admin, I can enable response caching | P1 |

### Monitoring

| ID | Story | Priority |
|----|-------|----------|
| US-30 | As an admin, I can view usage metrics (queries, tokens, costs) | P1 |
| US-31 | As an admin, I can see error rates and latency | P1 |
| US-32 | As an admin, I can export analytics data | P2 |

---

## 4. Technical Architecture

### Monorepo Structure

```
llm-council/
├── package.json                     # Workspace root
├── bun.lockb
├── wrangler.toml                    # Cloudflare Workers config
├── .github/
│   └── workflows/
│       ├── ci.yml                   # Test on PR
│       ├── deploy-staging.yml       # Deploy to staging
│       └── deploy-production.yml    # Deploy to production
│
├── packages/
│   ├── shared/                      # @council/shared
│   │   ├── package.json
│   │   └── src/
│   │       ├── types/               # TypeScript interfaces
│   │       ├── constants/           # Shared constants
│   │       ├── validation/          # Zod schemas
│   │       └── utils/               # Pure utility functions
│   │
│   ├── backend/                     # @council/backend (Cloudflare Worker)
│   │   ├── package.json
│   │   ├── wrangler.toml
│   │   └── src/
│   │       ├── index.ts             # Hono app entry
│   │       ├── routes/
│   │       │   ├── auth.ts          # WorkOS auth routes
│   │       │   ├── conversations.ts # Conversation CRUD
│   │       │   ├── council.ts       # Council query endpoint
│   │       │   ├── flags.ts         # Feature flags API
│   │       │   └── health.ts        # Health check
│   │       ├── services/
│   │       │   ├── council.ts       # 3-stage logic
│   │       │   ├── openrouter.ts    # LLM API client
│   │       │   ├── storage.ts       # D1 database access
│   │       │   ├── flags.ts         # Feature flag service
│   │       │   └── auth.ts          # WorkOS integration
│   │       ├── middleware/
│   │       │   ├── auth.ts          # Auth middleware
│   │       │   ├── rateLimit.ts     # Rate limiting
│   │       │   └── errors.ts        # Error handling
│   │       └── lib/
│   │           ├── retry.ts         # Retry with backoff
│   │           └── logger.ts        # Structured logging
│   │
│   └── frontend/                    # @council/frontend (Cloudflare Pages)
│       ├── package.json
│       ├── astro.config.mjs         # Astro + Cloudflare adapter
│       └── src/
│           ├── layouts/
│           │   └── Layout.astro     # Base HTML layout
│           ├── pages/
│           │   ├── index.astro      # Landing/login (static)
│           │   ├── app.astro        # Chat app shell
│           │   └── admin/
│           │       └── flags.astro  # Feature flags admin
│           ├── components/
│           │   ├── islands/         # Interactive Preact components
│           │   │   ├── ChatInterface.tsx
│           │   │   ├── Stage1.tsx
│           │   │   ├── Stage2.tsx
│           │   │   ├── Stage3.tsx
│           │   │   └── FlagToggle.tsx
│           │   └── static/          # Static Astro components
│           │       ├── Header.astro
│           │       └── Footer.astro
│           ├── api/                 # API client utilities
│           ├── hooks/               # Preact hooks
│           └── stores/              # State management
│
├── migrations/                      # D1 migrations
│   ├── 0001_initial.sql
│   └── 0002_feature_flags.sql
│
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLOUDFLARE EDGE                                 │
│                                                                              │
│  ┌──────────────────┐         ┌──────────────────────────────────────────┐  │
│  │ Cloudflare Pages │         │         Cloudflare Workers               │  │
│  │                  │         │                                          │  │
│  │  ┌────────────┐  │  API    │  ┌─────────────────────────────────────┐ │  │
│  │  │   Astro    │──┼────────►│  │            Hono App                 │ │  │
│  │  │  + Preact  │  │         │  │                                     │ │  │
│  │  │  Islands   │  │         │  │  /api/auth/*    → WorkOS            │ │  │
│  │  │            │  │         │  │  /api/council/* → Council Service   │ │  │
│  │  │  - Static  │  │         │  │  /api/flags/*   → Flags Service     │ │  │
│  │  │  - Islands │◄─┼─────────┼──│  /api/convos/*  → Storage Service   │ │  │
│  │  └────────────┘  │   SSE   │  │                                     │ │  │
│  │                  │         │  └────────────────────┬────────────────┘ │  │
│  └──────────────────┘         │                       │                  │  │
│                               │                       ▼                  │  │
│                               │  ┌──────────────────────────────────────┐│  │
│                               │  │         Cloudflare D1               ││  │
│                               │  │         (SQLite Edge DB)            ││  │
│                               │  │                                      ││  │
│                               │  │  - users                            ││  │
│                               │  │  - conversations                    ││  │
│                               │  │  - messages                         ││  │
│                               │  │  - feature_flags                    ││  │
│                               │  │  - analytics                        ││  │
│                               │  └──────────────────────────────────────┘│  │
│                               └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ HTTPS
                                        ▼
                    ┌───────────────────────────────────────┐
                    │           External Services           │
                    │                                       │
                    │  ┌─────────────┐  ┌───────────────┐  │
                    │  │   WorkOS    │  │  OpenRouter   │  │
                    │  │   (Auth)    │  │  (LLM API)    │  │
                    │  └─────────────┘  └───────────────┘  │
                    └───────────────────────────────────────┘
```

### Technology Choices

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Runtime** | Cloudflare Workers | Edge deployment, 5-min CPU limit on paid |
| **Backend Framework** | Hono | 14KB, fast, Cloudflare-native |
| **Frontend Shell** | Astro | Zero JS static pages, Cloudflare-owned (Jan 2026) |
| **Interactive UI** | Preact (islands) | 3KB, React-compatible, hydrated on demand |
| **Database** | Cloudflare D1 | SQLite at edge, included in Workers plan |
| **Auth** | WorkOS | SSO-ready, generous free tier |
| **LLM Gateway** | OpenRouter | Multi-model, unified API |
| **Types** | TypeScript + Zod | Compile-time + runtime validation |

**Astro Islands Architecture:**

| Page | Rendering | JS Shipped | Notes |
|------|-----------|------------|-------|
| `/` (landing) | Static | 0KB | Pure HTML |
| `/login` | Static | ~1KB | Minimal form JS |
| `/app` | Island | ~15KB | Chat interface (Preact) |
| `/admin/flags` | Hybrid | ~5KB | Toggle components |

---

## 5. Feature Specifications

### 5.1 Council Query Flow

**Request:**
```typescript
POST /api/council/query
{
  "conversation_id": "uuid",  // Optional, creates new if omitted
  "query": "User's question"
}
```

**Response (SSE Stream):**
```
event: stage1_start
data: {}

event: stage1_complete
data: {"responses": [...]}

event: stage2_start
data: {}

event: stage2_complete
data: {"rankings": [...], "aggregate": [...]}

event: stage3_start
data: {}

event: stage3_complete
data: {"synthesis": {...}}

event: complete
data: {"conversation_id": "uuid"}
```

### 5.2 Model Tiers

**Premium Tier:**
```typescript
const PREMIUM_MODELS = {
  council: [
    'openai/gpt-5.1',
    'google/gemini-3-pro-preview',
    'anthropic/claude-sonnet-4.5',
    'x-ai/grok-4',
  ],
  chairman: 'google/gemini-3-pro-preview',
  title: 'google/gemini-2.5-flash',
};
```

**Budget Tier:**
```typescript
const BUDGET_MODELS = {
  council: [
    'openai/gpt-4o-mini',
    'google/gemini-2.0-flash',
    'anthropic/claude-3.5-haiku',
    'mistralai/mistral-small',
  ],
  chairman: 'google/gemini-2.0-flash',
  title: 'google/gemini-2.0-flash',
};
```

### 5.3 Processing Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Real-time + Streaming** | SSE streaming as stages complete | Interactive use |
| **Real-time + Batch Response** | Single response after all stages | Simple integration |
| **Cached** | Return cached response if query matches | Cost reduction |
| **Batch** | Queue for async processing (future) | Bulk analysis |

---

## 6. Feature Flags System

### Flag Categories

| Category | Flags |
|----------|-------|
| **Model Tier** | `model_tier`: `premium` \| `budget` |
| **Processing** | `streaming_enabled`: boolean<br>`caching_enabled`: boolean<br>`cache_ttl_hours`: number |
| **Council** | `council_size`: 3-5<br>`chairman_model`: model ID<br>`anonymization_enabled`: boolean |
| **UI** | `show_stage1_tab`: boolean<br>`show_stage2_tab`: boolean<br>`show_aggregate_rankings`: boolean |

### Flag Storage Schema

```sql
CREATE TABLE feature_flags (
  id TEXT PRIMARY KEY,
  key TEXT NOT NULL UNIQUE,
  value TEXT NOT NULL,  -- JSON encoded
  type TEXT NOT NULL,   -- 'string' | 'number' | 'boolean' | 'json'
  description TEXT,
  updated_at TEXT NOT NULL,
  updated_by TEXT REFERENCES users(id)
);

-- Default values
INSERT INTO feature_flags (id, key, value, type, description, updated_at) VALUES
  ('1', 'model_tier', '"budget"', 'string', 'premium or budget model set', datetime('now')),
  ('2', 'streaming_enabled', 'true', 'boolean', 'Enable SSE streaming', datetime('now')),
  ('3', 'caching_enabled', 'false', 'boolean', 'Cache identical queries', datetime('now')),
  ('4', 'council_size', '4', 'number', 'Number of council members', datetime('now')),
  ('5', 'anonymization_enabled', 'true', 'boolean', 'Anonymize responses in Stage 2', datetime('now'));
```

### Flag Service API

```typescript
// GET /api/flags
// Returns all flags for authenticated user

// PUT /api/flags/:key
// Update a single flag (admin only)
{
  "value": "premium"
}

// POST /api/flags/reset
// Reset all flags to defaults (admin only)
```

---

## 7. Authentication (WorkOS)

### Implementation

```typescript
// backend/src/services/auth.ts
import { WorkOS } from '@workos-inc/node';

const workos = new WorkOS(env.WORKOS_API_KEY);

export async function createAuthUrl(redirectUri: string): Promise<string> {
  return workos.userManagement.getAuthorizationUrl({
    provider: 'authkit',
    redirectUri,
    clientId: env.WORKOS_CLIENT_ID,
  });
}

export async function authenticateWithCode(code: string) {
  return workos.userManagement.authenticateWithCode({
    code,
    clientId: env.WORKOS_CLIENT_ID,
  });
}

export async function getUser(accessToken: string) {
  return workos.userManagement.getUser(accessToken);
}
```

### Auth Middleware

```typescript
// backend/src/middleware/auth.ts
import { Context, Next } from 'hono';
import { verifyToken } from '../services/auth';

export async function authMiddleware(c: Context, next: Next) {
  const token = c.req.header('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return c.json({ error: 'Unauthorized' }, 401);
  }

  try {
    const user = await verifyToken(token);
    c.set('user', user);
    await next();
  } catch {
    return c.json({ error: 'Invalid token' }, 401);
  }
}
```

### User Roles

| Role | Permissions |
|------|-------------|
| **user** | Submit queries, view own conversations |
| **admin** | All user permissions + manage feature flags + view analytics |

---

## 8. Database Schema (D1)

```sql
-- migrations/0001_initial.sql

-- Users (synced from WorkOS)
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  workos_id TEXT UNIQUE NOT NULL,
  email TEXT NOT NULL,
  name TEXT,
  role TEXT NOT NULL DEFAULT 'user',
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  last_login_at TEXT
);

-- Conversations
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  title TEXT DEFAULT 'New Conversation',
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_conversations_user ON conversations(user_id);

-- Messages
CREATE TABLE messages (
  id TEXT PRIMARY KEY,
  conversation_id TEXT NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  role TEXT NOT NULL,  -- 'user' | 'assistant'
  content TEXT,        -- For user messages
  stage1 TEXT,         -- JSON for assistant
  stage2 TEXT,         -- JSON for assistant
  stage3 TEXT,         -- JSON for assistant
  metadata TEXT,       -- JSON (label_to_model, aggregate_rankings)
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_messages_conversation ON messages(conversation_id);

-- Feature Flags (see section 6)
CREATE TABLE feature_flags (
  id TEXT PRIMARY KEY,
  key TEXT NOT NULL UNIQUE,
  value TEXT NOT NULL,
  type TEXT NOT NULL,
  description TEXT,
  updated_at TEXT NOT NULL,
  updated_by TEXT REFERENCES users(id)
);

-- Analytics
CREATE TABLE analytics (
  id TEXT PRIMARY KEY,
  user_id TEXT REFERENCES users(id),
  event_type TEXT NOT NULL,
  event_data TEXT,  -- JSON
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_analytics_user ON analytics(user_id);
CREATE INDEX idx_analytics_type ON analytics(event_type);
CREATE INDEX idx_analytics_date ON analytics(created_at);

-- Query Cache
CREATE TABLE query_cache (
  id TEXT PRIMARY KEY,
  query_hash TEXT NOT NULL UNIQUE,
  response TEXT NOT NULL,  -- JSON
  model_tier TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  expires_at TEXT NOT NULL
);

CREATE INDEX idx_cache_hash ON query_cache(query_hash);
CREATE INDEX idx_cache_expires ON query_cache(expires_at);
```

---

## 9. API Specification

### Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/health` | No | Health check |
| POST | `/api/auth/login` | No | Initiate WorkOS login |
| GET | `/api/auth/callback` | No | WorkOS callback |
| POST | `/api/auth/logout` | Yes | End session |
| GET | `/api/auth/me` | Yes | Get current user |
| GET | `/api/conversations` | Yes | List user's conversations |
| POST | `/api/conversations` | Yes | Create conversation |
| GET | `/api/conversations/:id` | Yes | Get conversation |
| DELETE | `/api/conversations/:id` | Yes | Delete conversation |
| POST | `/api/council/query` | Yes | Submit council query |
| GET | `/api/flags` | Yes | Get feature flags |
| PUT | `/api/flags/:key` | Admin | Update flag |
| GET | `/api/analytics` | Admin | Get analytics |

### Error Response Format

```typescript
interface ErrorResponse {
  error: {
    code: string;
    message: string;
    details?: Record<string, unknown>;
  };
  requestId: string;
}
```

### Rate Limits

| Tier | Limit |
|------|-------|
| Anonymous | 10 req/min |
| Authenticated | 60 req/min |
| Admin | 120 req/min |

---

## 10. Testing Strategy

### Test Pyramid

```
                    ▲
                   /|\
                  / | \        E2E Tests (10%)
                 /  |  \       - Critical user flows
                /   |   \      - Playwright
               /────┼────\
              /     |     \    Integration Tests (30%)
             /      |      \   - API endpoints
            /       |       \  - Database operations
           /────────┼────────\ - External service mocks
          /         |         \
         /          |          \  Unit Tests (60%)
        /           |           \ - Services
       /            |            \- Utils
      /─────────────┼─────────────\- Validators
```

### Unit Tests

```typescript
// tests/unit/services/council.test.ts
import { describe, test, expect, mock } from 'bun:test';
import { parseRankingFromText, calculateAggregateRankings } from '@council/backend/services/council';

describe('parseRankingFromText', () => {
  test('parses numbered list format', () => {
    const text = `
      FINAL RANKING:
      1. Response C
      2. Response A
      3. Response B
    `;
    expect(parseRankingFromText(text)).toEqual([
      'Response C',
      'Response A',
      'Response B'
    ]);
  });

  test('handles missing FINAL RANKING header', () => {
    const text = 'Response A is best, then B, then C';
    const result = parseRankingFromText(text);
    expect(result).toContain('Response A');
  });

  test('returns empty array for invalid input', () => {
    expect(parseRankingFromText('')).toEqual([]);
  });
});

describe('calculateAggregateRankings', () => {
  test('calculates average positions correctly', () => {
    const stage2Results = [
      { model: 'gpt', parsed_ranking: ['Response A', 'Response B'] },
      { model: 'claude', parsed_ranking: ['Response B', 'Response A'] },
    ];
    const labelToModel = {
      'Response A': 'model-a',
      'Response B': 'model-b',
    };

    const result = calculateAggregateRankings(stage2Results, labelToModel);

    expect(result[0].average_rank).toBe(1.5);
    expect(result[1].average_rank).toBe(1.5);
  });
});
```

### Integration Tests

```typescript
// tests/integration/api/council.test.ts
import { describe, test, expect, beforeAll, afterAll } from 'bun:test';
import { unstable_dev } from 'wrangler';

describe('Council API', () => {
  let worker: Awaited<ReturnType<typeof unstable_dev>>;

  beforeAll(async () => {
    worker = await unstable_dev('src/index.ts', {
      experimental: { disableExperimentalWarning: true },
    });
  });

  afterAll(async () => {
    await worker.stop();
  });

  test('POST /api/council/query returns SSE stream', async () => {
    const response = await worker.fetch('/api/council/query', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer test-token',
      },
      body: JSON.stringify({ query: 'Test question' }),
    });

    expect(response.status).toBe(200);
    expect(response.headers.get('content-type')).toContain('text/event-stream');
  });

  test('requires authentication', async () => {
    const response = await worker.fetch('/api/council/query', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ query: 'Test' }),
    });

    expect(response.status).toBe(401);
  });
});
```

### E2E Tests

```typescript
// tests/e2e/council-flow.test.ts
import { test, expect } from '@playwright/test';

test.describe('Council Query Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login
    await page.goto('/login');
    await page.fill('[name="email"]', 'test@example.com');
    await page.fill('[name="password"]', 'password');
    await page.click('button[type="submit"]');
    await page.waitForURL('/');
  });

  test('complete council query shows all stages', async ({ page }) => {
    // Submit query
    await page.fill('textarea', 'What is machine learning?');
    await page.click('button:has-text("Send")');

    // Wait for all stages
    await expect(page.locator('[data-testid="stage1"]')).toBeVisible({ timeout: 120000 });
    await expect(page.locator('[data-testid="stage2"]')).toBeVisible({ timeout: 120000 });
    await expect(page.locator('[data-testid="stage3"]')).toBeVisible({ timeout: 120000 });

    // Verify content
    await expect(page.locator('[data-testid="aggregate-rankings"]')).toContainText('#1');
  });

  test('can switch between stage tabs', async ({ page }) => {
    await page.goto('/conversations/existing-id');

    // Click through tabs
    await page.click('[data-tab="stage1"]');
    await expect(page.locator('[data-testid="stage1-content"]')).toBeVisible();

    await page.click('[data-tab="stage2"]');
    await expect(page.locator('[data-testid="stage2-content"]')).toBeVisible();
  });
});
```

### Test Coverage Requirements

| Category | Minimum Coverage |
|----------|------------------|
| Core services (council, auth) | 90% |
| API routes | 85% |
| Utilities | 80% |
| UI components | 70% |
| **Overall** | **80%** |

---

## 11. Deployment & CI/CD

### GitHub Actions Workflows

**CI (on Pull Request):**
```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun run typecheck

  test-unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun test tests/unit

  test-integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun test tests/integration
    env:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY_TEST }}

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun run build
```

**Deploy to Staging (on merge to main):**
```yaml
# .github/workflows/deploy-staging.yml
name: Deploy Staging

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun run build

      - name: Deploy Worker
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: deploy --env staging

      - name: Deploy Pages
        uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          projectName: llm-council-staging
          directory: packages/frontend/dist

  e2e-test:
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bunx playwright install --with-deps
      - run: bun run test:e2e
        env:
          BASE_URL: https://staging.llm-council.pages.dev
```

**Deploy to Production (manual trigger):**
```yaml
# .github/workflows/deploy-production.yml
name: Deploy Production

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Type "deploy" to confirm'
        required: true

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Validate confirmation
        if: github.event.inputs.confirm != 'deploy'
        run: |
          echo "Deployment not confirmed"
          exit 1

  deploy:
    needs: validate
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun run build

      - name: Deploy Worker
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: deploy --env production

      - name: Deploy Pages
        uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          projectName: llm-council
          directory: packages/frontend/dist
```

### Wrangler Configuration

```toml
# wrangler.toml
name = "llm-council-api"
main = "packages/backend/src/index.ts"
compatibility_date = "2024-01-01"

[env.staging]
name = "llm-council-api-staging"
vars = { ENVIRONMENT = "staging" }

[[env.staging.d1_databases]]
binding = "DB"
database_name = "llm-council-staging"
database_id = "xxx"

[env.production]
name = "llm-council-api"
vars = { ENVIRONMENT = "production" }

[[env.production.d1_databases]]
binding = "DB"
database_name = "llm-council"
database_id = "yyy"
```

---

## 12. Cost Analysis

### Infrastructure Costs

| Service | Free Tier | Paid Tier | Expected |
|---------|-----------|-----------|----------|
| **Cloudflare Workers** | 100K req/day | $5/mo (10M req) | Free |
| **Cloudflare Pages** | Unlimited | - | Free |
| **Cloudflare D1** | 5M reads, 100K writes/day | $0.75/M reads | Free |
| **WorkOS** | 1M MAU | $0.05/MAU | Free |
| **GitHub Actions** | 2000 min/mo | $0.008/min | Free |
| **Total Infrastructure** | | | **$0-5/mo** |

### LLM API Costs (Per Query)

| Tier | Est. Tokens | Cost/Query | 100 queries/mo |
|------|-------------|------------|----------------|
| **Premium** | 33,000 | $0.15-0.20 | $15-20 |
| **Budget** | 33,000 | $0.03-0.05 | $3-5 |

### Recommended Configuration

For $10/month budget:
- Infrastructure: $0 (free tiers)
- LLM costs: $10
- Queries: ~50-200/month depending on tier

---

## 13. Security Requirements

### Data Protection

| Requirement | Implementation |
|-------------|----------------|
| Encryption at rest | D1 default encryption |
| Encryption in transit | HTTPS enforced (Cloudflare) |
| API key storage | Cloudflare secrets (encrypted) |
| User data isolation | Row-level filtering by user_id |
| Session security | Secure, HttpOnly, SameSite cookies |

### Authentication Security

| Requirement | Implementation |
|-------------|----------------|
| Password hashing | Handled by WorkOS |
| Brute force protection | WorkOS + rate limiting |
| Session expiry | 7 days, refresh on activity |
| CSRF protection | SameSite cookies + token validation |

### API Security

| Requirement | Implementation |
|-------------|----------------|
| Rate limiting | Per-user, per-endpoint limits |
| Input validation | Zod schemas on all inputs |
| SQL injection | Parameterized queries only |
| XSS prevention | Response Content-Type headers |
| CORS | Whitelist allowed origins |

### Audit Logging

```typescript
// Log security events
interface SecurityEvent {
  type: 'login' | 'logout' | 'failed_login' | 'flag_change' | 'data_access';
  user_id?: string;
  ip_address: string;
  user_agent: string;
  details: Record<string, unknown>;
  timestamp: string;
}
```

---

## 14. Migration Plan

### Phase 1: Foundation (Week 1)

- [ ] Initialize monorepo with Bun workspaces
- [ ] Set up shared types package
- [ ] Create D1 database schema
- [ ] Set up GitHub Actions CI

### Phase 2: Backend (Week 2)

- [ ] Port council service to Hono
- [ ] Implement D1 storage layer
- [ ] Add WorkOS authentication
- [ ] Implement feature flags service
- [ ] Write unit tests (80% coverage)

### Phase 3: Frontend (Week 3)

- [ ] Port React components to Preact
- [ ] Implement auth UI (login, signup)
- [ ] Add feature flags admin UI
- [ ] Write component tests

### Phase 4: Integration (Week 4)

- [ ] Integration testing
- [ ] E2E test suite
- [ ] Deploy to staging
- [ ] Manual QA testing

### Phase 5: Production (Week 5)

- [ ] Data migration script
- [ ] Production deployment
- [ ] Monitoring setup
- [ ] Documentation update

### Data Migration

```typescript
// scripts/migrate-data.ts
// Migrate JSON files to D1

import { readdir, readFile } from 'fs/promises';

async function migrate(db: D1Database) {
  const files = await readdir('data/conversations');

  for (const file of files) {
    const content = await readFile(`data/conversations/${file}`, 'utf-8');
    const conversation = JSON.parse(content);

    // Insert conversation
    await db.prepare(
      'INSERT INTO conversations (id, user_id, title, created_at) VALUES (?, ?, ?, ?)'
    ).bind(conversation.id, 'default-user', conversation.title, conversation.created_at).run();

    // Insert messages
    for (const message of conversation.messages) {
      await db.prepare(
        'INSERT INTO messages (id, conversation_id, role, content, stage1, stage2, stage3) VALUES (?, ?, ?, ?, ?, ?, ?)'
      ).bind(
        crypto.randomUUID(),
        conversation.id,
        message.role,
        message.content || null,
        JSON.stringify(message.stage1) || null,
        JSON.stringify(message.stage2) || null,
        JSON.stringify(message.stage3) || null,
      ).run();
    }
  }
}
```

---

## 15. Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Cloudflare Worker timeout | Medium | High | Use streaming, paid tier has 5-min limit |
| WorkOS rate limits | Low | Medium | Cache tokens, implement backoff |
| OpenRouter downtime | Medium | High | Graceful degradation, error messages |
| D1 query limits | Low | Medium | Optimize queries, use caching |
| Cost overrun (LLM) | Medium | Medium | Budget model default, usage alerts |
| Breaking changes (deps) | Low | Low | Lock dependency versions |

---

## 16. RAG System (Future)

> **Status**: Planned for v2.1+
> **Priority**: P2 (after auth, testing, and feature flags)

### 16.1 Overview

Implement a Retrieval-Augmented Generation (RAG) system to enable semantic search across:
- **Conversations**: Find past council deliberations by topic
- **Queries**: Search user questions for patterns and reuse
- **Notes**: Search admin annotations and conversation notes
- **Council Responses**: Search synthesized answers for knowledge retrieval

### 16.2 Architecture: Cloudflare-Native Stack

Leverage Cloudflare's integrated AI platform for minimal cost and latency:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Council of Elrond                            │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │   D1 (SQL)   │    │  Vectorize   │    │   Workers AI     │  │
│  │              │    │  (Vectors)   │    │  (Embeddings)    │  │
│  │ conversations│◄──►│              │◄──►│                  │  │
│  │ messages     │    │ conv_vectors │    │ @cf/baai/bge-m3  │  │
│  │ notes        │    │ query_vectors│    │                  │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
│         │                   │                     │             │
│         └───────────────────┼─────────────────────┘             │
│                             │                                   │
│                    ┌────────▼────────┐                          │
│                    │   RAG Service   │                          │
│                    │                 │                          │
│                    │ - Index on save │                          │
│                    │ - Search API    │                          │
│                    │ - Rerank results│                          │
│                    └─────────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

### 16.3 Recommended Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Vector DB** | Cloudflare Vectorize | Native integration, $0-6/mo for typical usage |
| **Embeddings** | Workers AI: `@cf/baai/bge-m3` | Multi-lingual, 1024-dim, $0.012/M tokens |
| **Reranking** | Workers AI: `@cf/baai/bge-reranker-base` | Improves retrieval accuracy |
| **Storage** | D1 (existing) | Document metadata, full-text backup |

### 16.4 Cost Analysis

**Cloudflare Vectorize Pricing:**

| Tier | Queried Dimensions | Stored Dimensions | Monthly Cost |
|------|-------------------|-------------------|--------------|
| Free | 30M included | 5M included | $0.00 |
| Paid | 50M included, then $0.01/M | 10M included, then $0.05/100M | ~$0.50-6.00 |

**Estimated Costs for Council of Elrond:**

| Scenario | Vectors | Queries/mo | Est. Cost |
|----------|---------|------------|-----------|
| Small (100 conversations) | 1,000 | 500 | **Free** |
| Medium (1,000 conversations) | 10,000 | 5,000 | **$0.10** |
| Large (10,000 conversations) | 100,000 | 50,000 | **$1.50** |
| Enterprise (100,000 conversations) | 1,000,000 | 500,000 | **$15.00** |

**Workers AI Embedding Costs:**

| Model | Cost/M Tokens | Neurons/Call | Notes |
|-------|---------------|--------------|-------|
| `bge-m3` | $0.012 | 1,075 | Best value, multi-lingual |
| `bge-base-en-v1.5` | $0.067 | 6,058 | English-only, 768-dim |
| `bge-large-en-v1.5` | $0.204 | 18,582 | Highest quality |

**Free Tier**: 10,000 Neurons/day = ~9 `bge-m3` calls or ~1.6 `bge-base` calls

### 16.5 Database Schema

```sql
-- migrations/0004_vectorize_metadata.sql

-- Track what's been indexed
CREATE TABLE vector_index_status (
  id TEXT PRIMARY KEY,
  entity_type TEXT NOT NULL,  -- 'conversation' | 'message' | 'note'
  entity_id TEXT NOT NULL,
  vector_id TEXT NOT NULL,    -- Vectorize vector ID
  indexed_at TEXT NOT NULL DEFAULT (datetime('now')),
  content_hash TEXT NOT NULL, -- Detect changes for re-indexing
  UNIQUE(entity_type, entity_id)
);

CREATE INDEX idx_vector_entity ON vector_index_status(entity_type, entity_id);
```

### 16.6 Vectorize Index Schema

```typescript
// Create indexes via Wrangler CLI
// wrangler vectorize create council-conversations --dimensions=1024 --metric=cosine

interface ConversationVector {
  id: string;           // conversation_id
  values: number[];     // 1024-dim embedding from bge-m3
  metadata: {
    title: string;
    user_id: string;
    created_at: string;
    message_count: number;
    has_notes: boolean;
  };
}

interface MessageVector {
  id: string;           // message_id
  values: number[];
  metadata: {
    conversation_id: string;
    role: 'user' | 'assistant';
    stage: 'query' | 'stage1' | 'stage2' | 'stage3';
    model?: string;     // For assistant messages
  };
}
```

### 16.7 RAG Service Implementation

```typescript
// backend/src/services/rag.ts

import { Ai } from '@cloudflare/ai';

interface Env {
  AI: Ai;
  VECTORIZE: VectorizeIndex;
  DB: D1Database;
}

export class RAGService {
  constructor(private env: Env) {}

  /**
   * Generate embedding for text using Workers AI
   */
  async embed(text: string): Promise<number[]> {
    const response = await this.env.AI.run('@cf/baai/bge-m3', {
      text: [text],
    });
    return response.data[0];
  }

  /**
   * Index a conversation (title + synthesis)
   */
  async indexConversation(conversationId: string): Promise<void> {
    const conv = await this.getConversationContent(conversationId);
    const embedding = await this.embed(conv.searchableText);

    await this.env.VECTORIZE.upsert([{
      id: conversationId,
      values: embedding,
      metadata: {
        title: conv.title,
        user_id: conv.user_id,
        created_at: conv.created_at,
        type: 'conversation',
      },
    }]);
  }

  /**
   * Semantic search across conversations
   */
  async search(query: string, options: SearchOptions = {}): Promise<SearchResult[]> {
    const queryEmbedding = await this.embed(query);

    const results = await this.env.VECTORIZE.query(queryEmbedding, {
      topK: options.limit || 10,
      filter: options.filter,
      returnMetadata: true,
    });

    // Optional: Rerank for better accuracy
    if (options.rerank) {
      return this.rerank(query, results);
    }

    return results.matches.map(m => ({
      id: m.id,
      score: m.score,
      metadata: m.metadata,
    }));
  }

  /**
   * Rerank results using bge-reranker
   */
  async rerank(query: string, results: VectorizeMatch[]): Promise<SearchResult[]> {
    const pairs = results.matches.map(m => ({
      query,
      context: m.metadata?.title || '',
    }));

    const scores = await this.env.AI.run('@cf/baai/bge-reranker-base', {
      pairs,
    });

    return results.matches
      .map((m, i) => ({ ...m, score: scores[i] }))
      .sort((a, b) => b.score - a.score);
  }
}
```

### 16.8 API Endpoints

```typescript
// New RAG endpoints

// Search conversations semantically
GET /api/search?q=machine+learning&limit=10
Response: { results: [{ id, title, score, snippet }] }

// Search within a conversation
GET /api/conversations/:id/search?q=neural+networks
Response: { results: [{ message_id, stage, score, snippet }] }

// Find similar conversations
GET /api/conversations/:id/similar?limit=5
Response: { results: [{ id, title, score }] }

// Admin: Reindex all content
POST /api/admin/reindex
Response: { indexed: 1523, errors: 0 }
```

### 16.9 User Stories

| ID | Story | Priority |
|----|-------|----------|
| US-40 | As a user, I can search my past conversations by topic | P1 |
| US-41 | As a user, I can find similar past queries before asking | P1 |
| US-42 | As a user, I can see related conversations while chatting | P2 |
| US-43 | As an admin, I can search all conversations across users | P1 |
| US-44 | As an admin, I can find conversations by notes content | P2 |

### 16.10 Alternative: Hybrid Search

For improved accuracy, consider hybrid search combining:

1. **Semantic (Vector)**: Conceptual similarity via embeddings
2. **Lexical (FTS)**: Exact keyword matching via D1 FTS5

```sql
-- D1 Full-Text Search (supplement to Vectorize)
CREATE VIRTUAL TABLE conversations_fts USING fts5(
  title,
  content,
  content='conversations',
  content_rowid='rowid'
);

-- Hybrid query: combine vector + FTS scores
```

### 16.11 Open Source Embedding Alternatives

If Workers AI costs become a concern, consider self-hosted alternatives:

| Model | Dimensions | Performance | Hosting |
|-------|------------|-------------|---------|
| `e5-small-v2` | 384 | 100% Top-5 accuracy | CPU-friendly |
| `nomic-embed-text-v1.5` | 768 | 71% accuracy | Apache 2.0 |
| `bge-m3` | 1024 | MTEB leader | Multi-lingual |

These can be deployed on:
- Cloudflare Workers (via ONNX runtime)
- Dedicated inference server
- HuggingFace Inference Endpoints

### 16.12 Implementation Phases

**Phase 1: Foundation**
- [ ] Create Vectorize index
- [ ] Add `vector_index_status` migration
- [ ] Implement RAGService with embed/search
- [ ] Index on conversation create/update

**Phase 2: Search UI**
- [ ] Add search bar to sidebar
- [ ] Display search results with snippets
- [ ] "Similar conversations" sidebar widget

**Phase 3: Advanced Features**
- [ ] Hybrid search (vector + FTS)
- [ ] Reranking for improved accuracy
- [ ] Conversation clustering/topics
- [ ] "Before you ask" similar query suggestions

---

## 17. Appendix

### A. Environment Variables

```bash
# Cloudflare
CLOUDFLARE_ACCOUNT_ID=xxx
CLOUDFLARE_API_TOKEN=xxx

# WorkOS
WORKOS_API_KEY=sk_xxx
WORKOS_CLIENT_ID=client_xxx

# OpenRouter
OPENROUTER_API_KEY=sk-or-xxx

# Application
ENVIRONMENT=production|staging|development
```

### B. NPM Scripts

```json
{
  "scripts": {
    "dev": "bun run --parallel dev:backend dev:frontend",
    "dev:backend": "wrangler dev packages/backend/src/index.ts",
    "dev:frontend": "astro dev --root packages/frontend",
    "build": "bun run build:shared && bun run --parallel build:backend build:frontend",
    "build:shared": "tsc -p packages/shared",
    "build:backend": "wrangler deploy --dry-run",
    "build:frontend": "astro build --root packages/frontend",
    "preview": "astro preview --root packages/frontend",
    "test": "bun test",
    "test:unit": "bun test tests/unit",
    "test:integration": "bun test tests/integration",
    "test:e2e": "playwright test",
    "lint": "eslint packages",
    "typecheck": "astro check && tsc --noEmit",
    "db:migrate": "wrangler d1 migrations apply",
    "db:seed": "bun run scripts/seed.ts"
  }
}
```

### C. Related Documentation

- [Architecture](./architecture.md)
- [Migration Steps](./migration-steps.md)
- [Framework Comparison](./framework-comparison.md)
- [Decision Making Process](./decision-making-process.md)
- [Quality Assurance](./quality-assurance.md)
- [CandidateAudit Extensions](./candidateaudit-extensions.md)
- [Cloudflare Research](./council-of-elrond-research-setup.md)

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Jan 2026 | Claude | Initial PRD |
| 1.1 | Jan 2026 | Claude | Updated frontend to Astro + Preact islands (Cloudflare acquisition) |
| 1.2 | Jan 2026 | Claude | Added Section 16: RAG System (Future) with Cloudflare Vectorize + Workers AI |
