# LLM Council Architecture: Cloudflare Edge + Astro + Hono

## Overview

This document describes the architecture for LLM Council v2.0 — a Cloudflare-native deployment using Astro (frontend) and Hono (backend) with D1 database.

> **Note:** Cloudflare acquired Astro in January 2026, making this stack first-party supported.

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Runtime** | Cloudflare Workers | Edge compute, 5-min CPU limit |
| **Backend** | Hono | Lightweight web framework with SSE |
| **Frontend Shell** | Astro | Static pages, zero JS by default |
| **Interactive UI** | Preact (islands) | Minimal React-compatible components |
| **Database** | Cloudflare D1 | SQLite at edge |
| **Auth** | WorkOS | Email/password + future SSO |
| **Markdown** | marked + DOMPurify | Render LLM responses safely |
| **Types** | TypeScript + Zod | Shared types, runtime validation |

---

## Monorepo Structure

```
llm-council/
├── package.json                    # Bun workspace root
├── bun.lockb
├── wrangler.toml                   # Root Cloudflare config
├── .env                            # Local secrets (gitignored)
├── .github/
│   └── workflows/
│       ├── ci.yml                  # Test on PR
│       ├── deploy-staging.yml      # Auto-deploy main → staging
│       └── deploy-production.yml   # Manual production deploy
│
├── packages/
│   │
│   ├── shared/                     # @council/shared
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts            # Barrel export
│   │       ├── types/
│   │       │   ├── api.ts          # Request/response types
│   │       │   ├── conversation.ts # Conversation models
│   │       │   ├── council.ts      # Stage result types
│   │       │   └── flags.ts        # Feature flag types
│   │       ├── constants/
│   │       │   └── models.ts       # Model tiers (premium/budget)
│   │       └── validation/
│   │           └── schemas.ts      # Zod schemas
│   │
│   ├── backend/                    # @council/backend (Cloudflare Worker)
│   │   ├── package.json
│   │   ├── wrangler.toml           # Worker-specific config
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts            # Hono app entry
│   │       ├── routes/
│   │       │   ├── auth.ts         # WorkOS authentication
│   │       │   ├── conversations.ts
│   │       │   ├── council.ts      # Main query endpoint
│   │       │   ├── flags.ts        # Feature flags API
│   │       │   └── health.ts
│   │       ├── services/
│   │       │   ├── council.ts      # 3-stage deliberation
│   │       │   ├── openrouter.ts   # LLM API client
│   │       │   ├── storage.ts      # D1 database access
│   │       │   ├── flags.ts        # Feature flag service
│   │       │   └── auth.ts         # WorkOS integration
│   │       ├── middleware/
│   │       │   ├── auth.ts         # JWT validation
│   │       │   ├── rateLimit.ts
│   │       │   └── errors.ts
│   │       └── lib/
│   │           ├── retry.ts        # Exponential backoff
│   │           └── logger.ts       # Structured logging
│   │
│   └── frontend/                   # @council/frontend (Cloudflare Pages)
│       ├── package.json
│       ├── astro.config.mjs        # Astro + @astrojs/cloudflare
│       ├── tsconfig.json
│       └── src/
│           ├── env.d.ts            # Astro type definitions
│           ├── layouts/
│           │   └── Layout.astro    # Base HTML shell
│           ├── pages/
│           │   ├── index.astro     # Landing page (static)
│           │   ├── login.astro     # Auth page (static + minimal JS)
│           │   ├── app.astro       # Chat application (island)
│           │   └── admin/
│           │       └── flags.astro # Feature flags UI (island)
│           ├── components/
│           │   ├── islands/        # Interactive Preact components
│           │   │   ├── ChatInterface.tsx
│           │   │   ├── Stage1.tsx
│           │   │   ├── Stage2.tsx
│           │   │   ├── Stage3.tsx
│           │   │   ├── Sidebar.tsx
│           │   │   └── FlagToggle.tsx
│           │   └── static/         # Non-interactive Astro components
│           │       ├── Header.astro
│           │       ├── Footer.astro
│           │       └── Logo.astro
│           ├── api/
│           │   └── client.ts       # Typed API client
│           ├── hooks/
│           │   ├── useConversation.ts
│           │   └── useSSE.ts       # SSE streaming hook
│           └── styles/
│               ├── global.css
│               └── markdown.css
│
├── migrations/                     # D1 database migrations
│   ├── 0001_initial.sql
│   ├── 0002_feature_flags.sql
│   └── 0003_analytics.sql
│
├── scripts/
│   ├── seed.ts                     # Database seeding
│   └── migrate-json.ts             # Legacy JSON → D1 migration
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
└── docs/
    ├── architecture.md             # This file
    ├── PRD.md                      # Product requirements
    ├── framework-comparison.md
    ├── migration-steps.md
    ├── decision-making-process.md
    ├── quality-assurance.md
    └── candidateaudit-extensions.md
```

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLOUDFLARE EDGE                                 │
│                                                                              │
│  ┌──────────────────────┐       ┌────────────────────────────────────────┐  │
│  │   Cloudflare Pages   │       │          Cloudflare Workers            │  │
│  │                      │       │                                        │  │
│  │  ┌────────────────┐  │       │  ┌──────────────────────────────────┐  │  │
│  │  │     Astro      │  │ API   │  │           Hono App               │  │  │
│  │  │                │──┼──────►│  │                                  │  │  │
│  │  │  Static Pages: │  │       │  │  /api/auth/*    → WorkOS        │  │  │
│  │  │  - / (landing) │  │       │  │  /api/council/* → Council Svc   │  │  │
│  │  │  - /login      │  │       │  │  /api/flags/*   → Flags Svc     │  │  │
│  │  │                │  │       │  │  /api/convos/*  → Storage Svc   │  │  │
│  │  │  Islands:      │◄─┼───────┼──│                                  │  │  │
│  │  │  - /app (chat) │  │  SSE  │  └────────────────┬─────────────────┘  │  │
│  │  │  - /admin      │  │       │                   │                    │  │
│  │  └────────────────┘  │       │                   ▼                    │  │
│  │                      │       │  ┌──────────────────────────────────┐  │  │
│  │  Preact islands:     │       │  │        Cloudflare D1            │  │  │
│  │  - ChatInterface     │       │  │        (SQLite Edge)            │  │  │
│  │  - Stage1/2/3        │       │  │                                  │  │  │
│  │  - FlagToggle        │       │  │  Tables:                        │  │  │
│  │                      │       │  │  - users                        │  │  │
│  └──────────────────────┘       │  │  - conversations                │  │  │
│                                 │  │  - messages                     │  │  │
│                                 │  │  - feature_flags                │  │  │
│                                 │  │  - analytics                    │  │  │
│                                 │  └──────────────────────────────────┘  │  │
│                                 └────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         │ HTTPS
                                         ▼
                     ┌───────────────────────────────────────┐
                     │          External Services            │
                     │                                       │
                     │  ┌─────────────┐  ┌───────────────┐  │
                     │  │   WorkOS    │  │  OpenRouter   │  │
                     │  │   (Auth)    │  │  (LLM API)    │  │
                     │  └─────────────┘  └───────────────┘  │
                     └───────────────────────────────────────┘
```

---

## Package Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│                        ROOT WORKSPACE                           │
│                         package.json                            │
│                    "workspaces": ["packages/*"]                 │
└─────────────────────────────────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌───────────────┐    ┌─────────────────┐    ┌─────────────────┐
│    shared     │    │     backend     │    │    frontend     │
│   @council/   │◄───│    @council/    │    │    @council/    │
│    shared     │    │     backend     │    │    frontend     │
├───────────────┤    ├─────────────────┤    ├─────────────────┤
│ TypeScript    │    │ Hono            │    │ Astro           │
│ Zod schemas   │◄───│ Cloudflare      │    │ Preact islands  │
│ Constants     │    │ Worker          │    │ Cloudflare      │
│ No runtime    │    │                 │    │ Pages           │
└───────────────┘    └─────────────────┘    └─────────────────┘
        ▲                                           │
        │                                           │
        └───────────────────────────────────────────┘
                    imports types from
```

---

## Astro Islands Architecture

The frontend uses Astro's islands architecture to minimize JavaScript:

```
Page Request
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Astro Build                               │
│                                                              │
│  Static Pages (0 KB JS)          Islands (hydrated)          │
│  ┌─────────────────────┐         ┌─────────────────────┐    │
│  │ index.astro         │         │ ChatInterface.tsx   │    │
│  │ login.astro         │         │ Stage1.tsx          │    │
│  │ Layout.astro        │         │ Stage2.tsx          │    │
│  │ Header.astro        │         │ Stage3.tsx          │    │
│  │ Footer.astro        │         │ FlagToggle.tsx      │    │
│  └─────────────────────┘         └─────────────────────┘    │
│          │                                │                  │
│          ▼                                ▼                  │
│    Pure HTML/CSS               Preact + client:load          │
│    No JavaScript               ~15KB hydrated                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Page-level breakdown:**

| Route | Type | JS Shipped | Description |
|-------|------|------------|-------------|
| `/` | Static | 0KB | Landing page |
| `/login` | Static | ~1KB | Auth form (minimal JS) |
| `/app` | Island | ~15KB | Chat interface (Preact) |
| `/admin/flags` | Hybrid | ~5KB | Toggle components |

---

## Data Flow (Council Query)

```
┌─────────────┐     ┌─────────────────────────────────────────────────┐
│   Browser   │     │              CLOUDFLARE WORKER                   │
│   (Preact)  │     │                                                  │
└──────┬──────┘     │  ┌─────────────────────────────────────────┐    │
       │            │  │         routes/council.ts                │    │
       │ POST       │  │  POST /api/council/query                 │    │
       │ + SSE      │  └──────────────────┬──────────────────────┘    │
       ▼            │                     │                           │
┌──────────────┐    │                     ▼                           │
│ ChatInterface│    │  ┌─────────────────────────────────────────┐    │
│              │    │  │       services/council.ts                │    │
│  useSSE()   │◄──►│  │                                          │    │
│    │         │    │  │  stage1CollectResponses()                │    │
│    ├─► Stage1│    │  │         │                                │    │
│    ├─► Stage2│    │  │         ▼                                │    │
│    └─► Stage3│    │  │  ┌──────────────────────────────────┐   │    │
│              │    │  │  │    services/openrouter.ts        │   │    │
└──────────────┘    │  │  │    queryModelsParallel()         │──►│────┼───► OpenRouter
                    │  │  │    [GPT, Gemini, Claude, Grok]   │   │    │     (4 parallel)
                    │  │  └──────────────────────────────────┘   │    │
                    │  │         │                                │    │
                    │  │         ▼                                │    │
                    │  │  stage2CollectRankings()                │    │
                    │  │  (anonymize → rank → parse → aggregate) │    │
                    │  │         │                                │    │
                    │  │         ▼                                │    │
                    │  │  stage3SynthesizeFinal()                │    │
                    │  │  (chairman synthesis)                   │    │
                    │  │         │                                │    │
                    │  │         ▼                                │    │
                    │  │  ┌──────────────────────────────────┐   │    │
                    │  │  │     services/storage.ts          │   │    │
                    │  │  │     D1 database persistence      │───┼────┼───► Cloudflare D1
                    │  │  └──────────────────────────────────┘   │    │
                    │  └─────────────────────────────────────────┘    │
                    └─────────────────────────────────────────────────┘
```

---

## SSE Streaming Flow

```
Frontend (Preact)                     Backend (Hono Worker)
─────────────────                     ────────────────────

POST /api/council/query ──────────►   Receive request
                                      │
                                      ├─► stream.writeSSE({ event: 'stage1_start' })
◄─────────────────────────────────────┤
Show "Collecting responses..."        │
                                      ├─► await stage1CollectResponses()
                                      │
                                      ├─► stream.writeSSE({ event: 'stage1_complete', data })
◄─────────────────────────────────────┤
Render Stage1 tabs                    │
                                      ├─► stream.writeSSE({ event: 'stage2_start' })
◄─────────────────────────────────────┤
Show "Collecting rankings..."         │
                                      ├─► await stage2CollectRankings()
                                      │
                                      ├─► stream.writeSSE({ event: 'stage2_complete', data })
◄─────────────────────────────────────┤
Render Stage2 + aggregate rankings    │
                                      ├─► stream.writeSSE({ event: 'stage3_start' })
◄─────────────────────────────────────┤
Show "Synthesizing..."                │
                                      ├─► await stage3SynthesizeFinal()
                                      │
                                      ├─► stream.writeSSE({ event: 'stage3_complete', data })
◄─────────────────────────────────────┤
Render final answer                   │
                                      ├─► stream.writeSSE({ event: 'complete' })
◄─────────────────────────────────────┤
Mark loading complete                 Stream ends
```

---

## Development Workflow

### Quick Start

```bash
# Install dependencies (Bun workspaces)
bun install

# Start local development
bun run dev

# Access:
# - Frontend: http://localhost:4321 (Astro dev server)
# - Backend: http://localhost:8787 (Wrangler dev server)
```

### Individual Commands

```bash
# Frontend (Astro)
bun run dev:frontend     # Astro dev server

# Backend (Hono + Wrangler)
bun run dev:backend      # Wrangler local dev with D1

# Type checking
bun run typecheck        # Astro check + tsc

# Build
bun run build            # Build all packages

# Database
bun run db:migrate       # Apply D1 migrations
bun run db:seed          # Seed test data
```

### Package Scripts

**Root package.json:**
```json
{
  "workspaces": ["packages/*"],
  "scripts": {
    "dev": "bun run --parallel dev:backend dev:frontend",
    "dev:backend": "wrangler dev packages/backend/src/index.ts --local",
    "dev:frontend": "astro dev --root packages/frontend",
    "build": "bun run build:shared && bun run --parallel build:backend build:frontend",
    "build:shared": "tsc -p packages/shared",
    "build:backend": "wrangler deploy --dry-run",
    "build:frontend": "astro build --root packages/frontend",
    "preview": "astro preview --root packages/frontend",
    "typecheck": "astro check && tsc --noEmit",
    "db:migrate": "wrangler d1 migrations apply llm-council",
    "db:seed": "bun run scripts/seed.ts"
  }
}
```

---

## Deployment

### Cloudflare Pages + Workers

```bash
# Deploy to staging (auto on push to main)
git push origin main

# Deploy to production (manual)
# Go to GitHub Actions → Deploy Production → Run workflow
```

### Wrangler Configuration

```toml
# wrangler.toml
name = "llm-council-api"
main = "packages/backend/src/index.ts"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "llm-council"
database_id = "your-database-id"

[vars]
ENVIRONMENT = "development"

# Secrets (set via wrangler secret put):
# - OPENROUTER_API_KEY
# - WORKOS_API_KEY
# - WORKOS_CLIENT_ID
```

### Astro Configuration

```javascript
// astro.config.mjs
import { defineConfig } from 'astro/config';
import cloudflare from '@astrojs/cloudflare';
import preact from '@astrojs/preact';

export default defineConfig({
  output: 'hybrid',  // Static by default, SSR opt-in
  adapter: cloudflare(),
  integrations: [preact()],
});
```

---

## Environment Variables

| Variable | Required | Where | Description |
|----------|----------|-------|-------------|
| `OPENROUTER_API_KEY` | Yes | Worker secret | LLM API key |
| `WORKOS_API_KEY` | Yes | Worker secret | Auth API key |
| `WORKOS_CLIENT_ID` | Yes | Worker secret | Auth client ID |
| `ENVIRONMENT` | No | wrangler.toml | staging/production |

**Local development (.env):**
```bash
OPENROUTER_API_KEY=sk-or-v1-...
WORKOS_API_KEY=sk_...
WORKOS_CLIENT_ID=client_...
```

---

## Bundle Size Targets

| Package | Target | Contents |
|---------|--------|----------|
| shared | ~0KB (types only) | TypeScript interfaces, Zod schemas |
| backend | ~50KB | Hono + services (runs on Worker) |
| frontend (static) | 0KB | Landing, login pages |
| frontend (islands) | <15KB gzipped | Preact + chat components |

**Frontend island breakdown:**
- Preact: ~3KB
- marked: ~5KB
- DOMPurify: ~3KB
- App code: ~4KB
- **Total: ~15KB gzipped** (only on interactive pages)

---

## Related Documentation

- [PRD](./PRD.md) — Full product requirements
- [Framework Comparison](./framework-comparison.md) — Technology decisions
- [Migration Steps](./migration-steps.md) — Migration guide
- [Decision Making Process](./decision-making-process.md) — Council logic
- [Quality Assurance](./quality-assurance.md) — Testing strategy
- [CandidateAudit Extensions](./candidateaudit-extensions.md) — Batch audit integration
