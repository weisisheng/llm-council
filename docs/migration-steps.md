# LLM Council Migration Guide: Python → Bun/Hono/Preact

## Overview

This document provides step-by-step instructions for migrating LLM Council from Python/FastAPI + React to a Bun-based monorepo with Hono backend and Preact frontend.

---

## Pre-Migration Checklist

- [ ] Bun installed (`curl -fsSL https://bun.sh/install | bash`)
- [ ] Node.js 18+ installed (for Vite compatibility)
- [ ] Existing codebase backed up or committed to git
- [ ] OpenRouter API key available

---

## Phase 1: Foundation Setup (No Breaking Changes)

### Step 1.1: Initialize Bun Workspace

Create root `package.json`:

```json
{
  "name": "llm-council",
  "version": "1.0.0",
  "private": true,
  "workspaces": ["packages/*"],
  "scripts": {
    "dev": "bun run --parallel dev:backend dev:frontend",
    "dev:backend": "bun run --cwd packages/backend dev",
    "dev:frontend": "bun run --cwd packages/frontend dev",
    "build": "bun run build:shared && bun run --parallel build:backend build:frontend",
    "build:shared": "bun run --cwd packages/shared build",
    "build:backend": "bun run --cwd packages/backend build",
    "build:frontend": "bun run --cwd packages/frontend build",
    "typecheck": "bun run --parallel typecheck:shared typecheck:backend typecheck:frontend"
  },
  "devDependencies": {
    "typescript": "^5.4.0"
  }
}
```

Create directory structure:
```bash
mkdir -p packages/shared/src/types
mkdir -p packages/backend/src/{routes,services,middleware}
mkdir -p packages/frontend/src/components
```

### Step 1.2: Create Shared Types Package

**packages/shared/package.json:**
```json
{
  "name": "@llm-council/shared",
  "version": "1.0.0",
  "private": true,
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
    "typecheck": "tsc --noEmit",
    "dev": "tsc --watch"
  },
  "devDependencies": {
    "typescript": "^5.4.0"
  }
}
```

**packages/shared/tsconfig.json:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"]
}
```

**packages/shared/src/types/council.ts:**
```typescript
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

export interface AggregateRanking {
  model: string;
  average_rank: number;
  rankings_count: number;
}

export interface LabelToModel {
  [label: string]: string;
}

export interface CouncilMetadata {
  label_to_model: LabelToModel;
  aggregate_rankings: AggregateRanking[];
}
```

**packages/shared/src/types/conversation.ts:**
```typescript
import type { Stage1Result, Stage2Result, Stage3Result } from './council';

export interface UserMessage {
  role: 'user';
  content: string;
}

export interface AssistantMessage {
  role: 'assistant';
  stage1: Stage1Result[];
  stage2: Stage2Result[];
  stage3: Stage3Result;
}

export type Message = UserMessage | AssistantMessage;

export interface Conversation {
  id: string;
  created_at: string;
  title: string;
  messages: Message[];
}

export interface ConversationMetadata {
  id: string;
  created_at: string;
  title: string;
  message_count: number;
}
```

**packages/shared/src/types/api.ts:**
```typescript
import type { Stage1Result, Stage2Result, Stage3Result, CouncilMetadata } from './council';

export interface SendMessageRequest {
  content: string;
}

export interface SendMessageResponse {
  stage1: Stage1Result[];
  stage2: Stage2Result[];
  stage3: Stage3Result;
  metadata: CouncilMetadata;
}

export type SSEEventType =
  | 'stage1_start'
  | 'stage1_complete'
  | 'stage2_start'
  | 'stage2_complete'
  | 'stage3_start'
  | 'stage3_complete'
  | 'title_complete'
  | 'complete'
  | 'error';

export interface SSEEvent {
  type: SSEEventType;
  data?: unknown;
  metadata?: CouncilMetadata;
  message?: string;
}
```

**packages/shared/src/index.ts:**
```typescript
export * from './types/council';
export * from './types/conversation';
export * from './types/api';
```

### Step 1.3: Build and Verify Types

```bash
cd packages/shared
bun run build
# Should create dist/ with .js and .d.ts files
```

---

## Phase 2: Backend Migration

### Step 2.1: Create Backend Package

**packages/backend/package.json:**
```json
{
  "name": "@llm-council/backend",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "bun --watch src/index.ts",
    "build": "bun build src/index.ts --outdir ./dist --target bun",
    "start": "bun dist/index.js",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@llm-council/shared": "workspace:*",
    "hono": "^4.4.0"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "typescript": "^5.4.0"
  }
}
```

### Step 2.2: Port Configuration

**packages/backend/src/config.ts:**
```typescript
// Equivalent of backend/config.py

export const config = {
  OPENROUTER_API_KEY: process.env.OPENROUTER_API_KEY || '',
  OPENROUTER_API_URL: 'https://openrouter.ai/api/v1/chat/completions',

  COUNCIL_MODELS: [
    'openai/gpt-5.1',
    'google/gemini-3-pro-preview',
    'anthropic/claude-sonnet-4.5',
    'x-ai/grok-4',
  ],

  CHAIRMAN_MODEL: 'google/gemini-3-pro-preview',
  TITLE_MODEL: 'google/gemini-2.5-flash',

  DATA_DIR: process.env.DATA_DIR || 'data/conversations',
  PORT: Number(process.env.PORT) || 8001,
};
```

### Step 2.3: Port OpenRouter Client

**packages/backend/src/services/openrouter.ts:**
```typescript
// Equivalent of backend/openrouter.py

import { config } from '../config';

interface Message {
  role: 'user' | 'assistant' | 'system';
  content: string;
}

interface ModelResponse {
  content: string;
  reasoningDetails?: string;
}

export async function queryModel(
  model: string,
  messages: Message[],
  timeout = 120000
): Promise<ModelResponse | null> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(config.OPENROUTER_API_URL, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${config.OPENROUTER_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ model, messages }),
      signal: controller.signal,
    });

    clearTimeout(timeoutId);

    if (!response.ok) {
      console.error(`Error querying ${model}: ${response.status}`);
      return null;
    }

    const data = await response.json();
    const message = data.choices?.[0]?.message;

    return {
      content: message?.content || '',
      reasoningDetails: message?.reasoning_details,
    };
  } catch (error) {
    clearTimeout(timeoutId);
    console.error(`Error querying ${model}:`, error);
    return null;
  }
}

export async function queryModelsParallel(
  models: string[],
  messages: Message[]
): Promise<Map<string, ModelResponse | null>> {
  // Equivalent of asyncio.gather()
  const results = await Promise.all(
    models.map(async (model) => {
      const response = await queryModel(model, messages);
      return [model, response] as const;
    })
  );

  return new Map(results);
}
```

### Step 2.4: Port Storage Module

**packages/backend/src/services/storage.ts:**
```typescript
// Equivalent of backend/storage.py

import { mkdir, readdir, readFile, writeFile } from 'node:fs/promises';
import { existsSync } from 'node:fs';
import { join } from 'node:path';
import type {
  Conversation,
  ConversationMetadata,
  Stage1Result,
  Stage2Result,
  Stage3Result,
} from '@llm-council/shared';
import { config } from '../config';

async function ensureDataDir(): Promise<void> {
  if (!existsSync(config.DATA_DIR)) {
    await mkdir(config.DATA_DIR, { recursive: true });
  }
}

function getPath(id: string): string {
  return join(config.DATA_DIR, `${id}.json`);
}

export async function createConversation(id: string): Promise<Conversation> {
  await ensureDataDir();

  const conversation: Conversation = {
    id,
    created_at: new Date().toISOString(),
    title: 'New Conversation',
    messages: [],
  };

  await writeFile(getPath(id), JSON.stringify(conversation, null, 2));
  return conversation;
}

export async function getConversation(id: string): Promise<Conversation | null> {
  const path = getPath(id);
  if (!existsSync(path)) return null;

  const content = await readFile(path, 'utf-8');
  return JSON.parse(content);
}

export async function saveConversation(conversation: Conversation): Promise<void> {
  await ensureDataDir();
  await writeFile(getPath(conversation.id), JSON.stringify(conversation, null, 2));
}

export async function listConversations(): Promise<ConversationMetadata[]> {
  await ensureDataDir();

  const files = await readdir(config.DATA_DIR);
  const conversations: ConversationMetadata[] = [];

  for (const file of files) {
    if (!file.endsWith('.json')) continue;

    const content = await readFile(join(config.DATA_DIR, file), 'utf-8');
    const data: Conversation = JSON.parse(content);

    conversations.push({
      id: data.id,
      created_at: data.created_at,
      title: data.title || 'New Conversation',
      message_count: data.messages.length,
    });
  }

  return conversations.sort(
    (a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime()
  );
}

export async function addUserMessage(id: string, content: string): Promise<void> {
  const conversation = await getConversation(id);
  if (!conversation) throw new Error(`Conversation ${id} not found`);

  conversation.messages.push({ role: 'user', content });
  await saveConversation(conversation);
}

export async function addAssistantMessage(
  id: string,
  stage1: Stage1Result[],
  stage2: Stage2Result[],
  stage3: Stage3Result
): Promise<void> {
  const conversation = await getConversation(id);
  if (!conversation) throw new Error(`Conversation ${id} not found`);

  conversation.messages.push({ role: 'assistant', stage1, stage2, stage3 });
  await saveConversation(conversation);
}

export async function updateConversationTitle(id: string, title: string): Promise<void> {
  const conversation = await getConversation(id);
  if (!conversation) throw new Error(`Conversation ${id} not found`);

  conversation.title = title;
  await saveConversation(conversation);
}
```

### Step 2.5: Port Council Logic

**packages/backend/src/services/council.ts:**
```typescript
// Equivalent of backend/council.py

import type {
  Stage1Result,
  Stage2Result,
  Stage3Result,
  AggregateRanking,
  LabelToModel,
  CouncilMetadata,
} from '@llm-council/shared';
import { config } from '../config';
import { queryModel, queryModelsParallel } from './openrouter';

export async function stage1CollectResponses(userQuery: string): Promise<Stage1Result[]> {
  const messages = [{ role: 'user' as const, content: userQuery }];
  const responses = await queryModelsParallel(config.COUNCIL_MODELS, messages);

  const results: Stage1Result[] = [];
  for (const [model, response] of responses) {
    if (response) {
      results.push({ model, response: response.content });
    }
  }

  return results;
}

export async function stage2CollectRankings(
  userQuery: string,
  stage1Results: Stage1Result[]
): Promise<{ rankings: Stage2Result[]; labelToModel: LabelToModel }> {
  // Create anonymous labels
  const labels = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
  const labelToModel: LabelToModel = {};
  const anonymizedResponses: string[] = [];

  stage1Results.forEach((result, i) => {
    const label = `Response ${labels[i]}`;
    labelToModel[label] = result.model;
    anonymizedResponses.push(`${label}:\n${result.response}`);
  });

  // Build ranking prompt
  const prompt = `You are evaluating responses to this question: "${userQuery}"

Here are the responses to evaluate:

${anonymizedResponses.join('\n\n---\n\n')}

Please:
1. Evaluate each response for accuracy, completeness, and insight
2. Provide your reasoning for each evaluation
3. End with "FINAL RANKING:" followed by a numbered list like:
   1. Response C
   2. Response A
   3. Response B
   etc.

Do not include any text after the ranking list.`;

  const messages = [{ role: 'user' as const, content: prompt }];
  const responses = await queryModelsParallel(config.COUNCIL_MODELS, messages);

  const rankings: Stage2Result[] = [];
  for (const [model, response] of responses) {
    if (response) {
      rankings.push({
        model,
        ranking: response.content,
        parsed_ranking: parseRankingFromText(response.content),
      });
    }
  }

  return { rankings, labelToModel };
}

export function parseRankingFromText(text: string): string[] {
  // Look for "FINAL RANKING:" section
  const rankingMatch = text.match(/FINAL RANKING:[\s\S]*$/i);
  const rankingSection = rankingMatch ? rankingMatch[0] : text;

  // Extract numbered list items
  const numberedPattern = /\d+\.\s*(Response [A-Z])/gi;
  const matches = [...rankingSection.matchAll(numberedPattern)];

  if (matches.length > 0) {
    return matches.map((m) => m[1]);
  }

  // Fallback: find any "Response X" patterns
  const fallbackPattern = /Response [A-Z]/gi;
  const fallbackMatches = rankingSection.match(fallbackPattern);

  return fallbackMatches || [];
}

export async function stage3SynthesizeFinal(
  userQuery: string,
  stage1Results: Stage1Result[],
  stage2Results: Stage2Result[]
): Promise<Stage3Result> {
  const stage1Text = stage1Results
    .map((r) => `Model: ${r.model}\nResponse: ${r.response}`)
    .join('\n\n---\n\n');

  const stage2Text = stage2Results
    .map((r) => `Model: ${r.model}\nRanking:\n${r.ranking}`)
    .join('\n\n---\n\n');

  const prompt = `You are the chairman of an LLM council. Your task is to synthesize the best possible answer based on the council's deliberation.

Original Question: ${userQuery}

STAGE 1 - Individual Responses:
${stage1Text}

STAGE 2 - Peer Rankings:
${stage2Text}

Based on all the above, provide a comprehensive, synthesized answer that incorporates the best insights from all responses, weighted by the peer rankings.`;

  const messages = [{ role: 'user' as const, content: prompt }];
  const response = await queryModel(config.CHAIRMAN_MODEL, messages);

  return {
    model: config.CHAIRMAN_MODEL,
    response: response?.content || 'Failed to generate synthesis.',
  };
}

export function calculateAggregateRankings(
  stage2Results: Stage2Result[],
  labelToModel: LabelToModel
): AggregateRanking[] {
  const modelRanks: Map<string, number[]> = new Map();

  // Initialize with all models
  for (const model of Object.values(labelToModel)) {
    modelRanks.set(model, []);
  }

  // Collect ranks
  for (const result of stage2Results) {
    result.parsed_ranking.forEach((label, position) => {
      const model = labelToModel[label];
      if (model) {
        modelRanks.get(model)?.push(position + 1); // 1-indexed rank
      }
    });
  }

  // Calculate averages
  const rankings: AggregateRanking[] = [];
  for (const [model, ranks] of modelRanks) {
    if (ranks.length > 0) {
      const average = ranks.reduce((a, b) => a + b, 0) / ranks.length;
      rankings.push({
        model,
        average_rank: Math.round(average * 100) / 100,
        rankings_count: ranks.length,
      });
    }
  }

  return rankings.sort((a, b) => a.average_rank - b.average_rank);
}

export async function generateConversationTitle(userQuery: string): Promise<string> {
  const prompt = `Generate a very short title (3-5 words) summarizing this question: "${userQuery}"

Return only the title, no quotes or punctuation.`;

  const messages = [{ role: 'user' as const, content: prompt }];
  const response = await queryModel(config.TITLE_MODEL, messages);

  let title = response?.content || 'New Conversation';
  title = title.replace(/^["']|["']$/g, '').trim();
  return title.slice(0, 50);
}

export async function runFullCouncil(userQuery: string): Promise<{
  stage1: Stage1Result[];
  stage2: Stage2Result[];
  stage3: Stage3Result;
  metadata: CouncilMetadata;
}> {
  const stage1 = await stage1CollectResponses(userQuery);

  if (stage1.length === 0) {
    return {
      stage1: [],
      stage2: [],
      stage3: { model: config.CHAIRMAN_MODEL, response: 'All models failed to respond.' },
      metadata: { label_to_model: {}, aggregate_rankings: [] },
    };
  }

  const { rankings: stage2, labelToModel } = await stage2CollectRankings(userQuery, stage1);
  const aggregateRankings = calculateAggregateRankings(stage2, labelToModel);
  const stage3 = await stage3SynthesizeFinal(userQuery, stage1, stage2);

  return {
    stage1,
    stage2,
    stage3,
    metadata: {
      label_to_model: labelToModel,
      aggregate_rankings: aggregateRankings,
    },
  };
}
```

### Step 2.6: Create Hono Routes

**packages/backend/src/index.ts:**
```typescript
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { streamSSE } from 'hono/streaming';
import { config } from './config';
import * as storage from './services/storage';
import {
  stage1CollectResponses,
  stage2CollectRankings,
  stage3SynthesizeFinal,
  calculateAggregateRankings,
  generateConversationTitle,
  runFullCouncil,
} from './services/council';

const app = new Hono();

// CORS
app.use('*', cors({
  origin: ['http://localhost:5173', 'http://localhost:3000'],
  allowMethods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowHeaders: ['Content-Type'],
}));

// Health check
app.get('/', (c) => c.json({ status: 'ok', service: 'LLM Council API' }));

// List conversations
app.get('/api/conversations', async (c) => {
  const conversations = await storage.listConversations();
  return c.json(conversations);
});

// Create conversation
app.post('/api/conversations', async (c) => {
  const id = crypto.randomUUID();
  const conversation = await storage.createConversation(id);
  return c.json(conversation);
});

// Get conversation
app.get('/api/conversations/:id', async (c) => {
  const conversation = await storage.getConversation(c.req.param('id'));
  if (!conversation) return c.json({ error: 'Not found' }, 404);
  return c.json(conversation);
});

// Send message (non-streaming)
app.post('/api/conversations/:id/message', async (c) => {
  const id = c.req.param('id');
  const { content } = await c.req.json<{ content: string }>();

  const conversation = await storage.getConversation(id);
  if (!conversation) return c.json({ error: 'Not found' }, 404);

  const isFirstMessage = conversation.messages.length === 0;
  await storage.addUserMessage(id, content);

  if (isFirstMessage) {
    const title = await generateConversationTitle(content);
    await storage.updateConversationTitle(id, title);
  }

  const result = await runFullCouncil(content);
  await storage.addAssistantMessage(id, result.stage1, result.stage2, result.stage3);

  return c.json(result);
});

// Send message (SSE streaming)
app.post('/api/conversations/:id/message/stream', async (c) => {
  const id = c.req.param('id');
  const { content } = await c.req.json<{ content: string }>();

  const conversation = await storage.getConversation(id);
  if (!conversation) return c.json({ error: 'Not found' }, 404);

  const isFirstMessage = conversation.messages.length === 0;

  return streamSSE(c, async (stream) => {
    try {
      await storage.addUserMessage(id, content);

      let titlePromise: Promise<string> | null = null;
      if (isFirstMessage) {
        titlePromise = generateConversationTitle(content);
      }

      // Stage 1
      await stream.writeSSE({ data: JSON.stringify({ type: 'stage1_start' }) });
      const stage1 = await stage1CollectResponses(content);
      await stream.writeSSE({ data: JSON.stringify({ type: 'stage1_complete', data: stage1 }) });

      // Stage 2
      await stream.writeSSE({ data: JSON.stringify({ type: 'stage2_start' }) });
      const { rankings: stage2, labelToModel } = await stage2CollectRankings(content, stage1);
      const aggregateRankings = calculateAggregateRankings(stage2, labelToModel);
      await stream.writeSSE({
        data: JSON.stringify({
          type: 'stage2_complete',
          data: stage2,
          metadata: { label_to_model: labelToModel, aggregate_rankings: aggregateRankings },
        }),
      });

      // Stage 3
      await stream.writeSSE({ data: JSON.stringify({ type: 'stage3_start' }) });
      const stage3 = await stage3SynthesizeFinal(content, stage1, stage2);
      await stream.writeSSE({ data: JSON.stringify({ type: 'stage3_complete', data: stage3 }) });

      // Title
      if (titlePromise) {
        const title = await titlePromise;
        await storage.updateConversationTitle(id, title);
        await stream.writeSSE({ data: JSON.stringify({ type: 'title_complete', data: { title } }) });
      }

      await storage.addAssistantMessage(id, stage1, stage2, stage3);
      await stream.writeSSE({ data: JSON.stringify({ type: 'complete' }) });
    } catch (error) {
      await stream.writeSSE({ data: JSON.stringify({ type: 'error', message: String(error) }) });
    }
  });
});

export default {
  port: config.PORT,
  fetch: app.fetch,
};
```

### Step 2.7: Test Backend

```bash
# Start new backend on test port
PORT=8002 bun run packages/backend/src/index.ts

# Test health check
curl http://localhost:8002/

# Compare with Python backend running on 8001
# Both should return identical responses
```

---

## Phase 3: Frontend Migration

### Step 3.1: Create Frontend Package

**packages/frontend/package.json:**
```json
{
  "name": "@llm-council/frontend",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@llm-council/shared": "workspace:*",
    "preact": "^10.22.0",
    "marked": "^12.0.0",
    "dompurify": "^3.1.0"
  },
  "devDependencies": {
    "@preact/preset-vite": "^2.8.0",
    "@types/dompurify": "^3.0.5",
    "typescript": "^5.4.0",
    "vite": "^5.3.0"
  }
}
```

**packages/frontend/vite.config.ts:**
```typescript
import { defineConfig } from 'vite';
import preact from '@preact/preset-vite';

export default defineConfig({
  plugins: [preact()],
  server: {
    port: 5173,
  },
});
```

### Step 3.2: Convert Components

Key changes for each component:

1. Change imports: `import { useState } from 'react'` → `import { useState } from 'preact/hooks'`
2. Rename files: `.jsx` → `.tsx`
3. Add type annotations for props
4. Replace `className` with `class` (optional, both work in Preact)

### Step 3.3: Create Markdown Component

**packages/frontend/src/components/Markdown.tsx:**
```typescript
import { marked } from 'marked';
import DOMPurify from 'dompurify';

interface Props {
  children: string;
}

export function Markdown({ children }: Props) {
  const html = DOMPurify.sanitize(marked.parse(children) as string);
  return (
    <div
      class="markdown-content"
      dangerouslySetInnerHTML={{ __html: html }}
    />
  );
}
```

### Step 3.4: Verify Frontend

```bash
# Install dependencies
bun install

# Start frontend
bun run dev:frontend

# Check bundle size
bun run build:frontend
ls -la packages/frontend/dist/assets/
```

---

## Phase 4: Cleanup

### Step 4.1: Remove Python Code

```bash
# Delete Python backend
rm -rf backend/
rm pyproject.toml
rm -f uv.lock
```

### Step 4.2: Update Documentation

Update `CLAUDE.md` with new architecture details.

### Step 4.3: Add Docker Support

Create `Dockerfile` and `docker-compose.yml` as specified in the architecture document.

### Step 4.4: Update start.sh

```bash
#!/bin/bash
echo "Starting LLM Council..."
bun run dev
```

---

## Verification Checklist

### Backend Parity
- [ ] `GET /` returns health status
- [ ] `GET /api/conversations` lists all conversations
- [ ] `POST /api/conversations` creates new conversation
- [ ] `GET /api/conversations/:id` returns conversation
- [ ] `POST /api/conversations/:id/message` returns stages + metadata
- [ ] `POST /api/conversations/:id/message/stream` emits correct SSE events
- [ ] Existing JSON files load correctly

### Frontend Parity
- [ ] Sidebar shows conversations
- [ ] New conversation can be created
- [ ] Messages send and receive correctly
- [ ] Stage 1 tabs work
- [ ] Stage 2 tabs + aggregates work
- [ ] Stage 3 displays
- [ ] Markdown renders properly

### Performance
- [ ] Frontend bundle < 15KB gzipped
- [ ] Backend starts in < 100ms
- [ ] `bun install` < 5 seconds
