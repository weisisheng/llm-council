# Framework Comparison: Technology Decisions for LLM Council

This document captures the research and rationale behind technology choices for the LLM Council refactoring.

---

## Runtime: Bun vs Node.js

### Comparison

| Factor | Node.js | Bun |
|--------|---------|-----|
| **Maturity** | 15+ years, battle-tested | ~2 years, production-ready |
| **Package install** | npm: ~30s for this project | bun: ~2s (10-25x faster) |
| **TypeScript** | Requires build step | Native execution |
| **Startup time** | ~100-200ms | ~10-50ms |
| **Runtime speed** | Baseline | ~2-3x faster |
| **Compatibility** | 100% npm packages | ~99% npm packages |
| **Ecosystem** | Largest | Growing, compatible |

### Decision: Bun

**Rationale:**
1. Single runtime for entire stack (no Python + Node)
2. Native TypeScript — no compilation step for backend
3. Built-in workspaces for monorepo
4. Faster development iteration (instant starts, fast installs)
5. Escape hatch: Hono code runs on Node.js if needed

**Risk mitigation:** Hono is portable — if Bun causes issues, switch to Node.js with minimal changes.

---

## Backend Framework Comparison

### Candidates Evaluated

| Framework | Bundle | Speed (req/s) | Runtime | Type Safety |
|-----------|--------|---------------|---------|-------------|
| **Express** | ~200KB | ~15,000 | Node | Manual |
| **Fastify** | ~300KB | ~45,000 | Node | JSON Schema |
| **Hono** | ~14KB | ~50,000 | Any | TypeScript |
| **Elysia** | ~20KB | ~55,000 | Bun only | Automatic (Eden) |
| **uWebSockets** | ~50KB | ~100,000 | Node | Manual |

*Benchmarks are approximate "hello world" — real performance depends on application logic.*

### Express
- **Pros:** Most middleware available, largest community, everyone knows it
- **Cons:** Slowest, callback-based origins, no built-in TypeScript

### Fastify
- **Pros:** Fast, good plugin system, JSON Schema validation
- **Cons:** Node-only, larger than Hono, more complex setup

### Hono
- **Pros:** Tiny (14KB), fast, runs everywhere (Node, Bun, Deno, Cloudflare Workers)
- **Cons:** Smaller ecosystem, manual type sharing with frontend

### Elysia
- **Pros:** Fastest, automatic end-to-end types via Eden (like tRPC)
- **Cons:** Bun-only (no Node.js fallback), newer/less mature

### uWebSockets.js
- **Pros:** Absolute fastest for high-throughput scenarios
- **Cons:** Low-level API, harder to use, overkill for this project

### Decision: Hono

**Rationale:**
1. Smallest bundle (14KB)
2. Excellent performance (~50K req/s)
3. Portable — runs on Node, Bun, Deno, edge
4. Built-in SSE streaming support (`hono/streaming`)
5. Clean, modern API similar to Express
6. Active development, growing community

**Why not Elysia?** Eden's automatic types are compelling, but Bun-only lock-in is risky. Hono provides a Node.js escape hatch.

**Note:** For this project, backend framework performance is irrelevant — the bottleneck is LLM API latency (seconds), not router overhead (microseconds).

---

## Frontend Framework Comparison

### Candidates Evaluated

| Framework | Size (gzip) | Migration | Reactivity |
|-----------|-------------|-----------|------------|
| **React 19** | ~45KB | None | setState |
| **Preact** | ~3KB | Minimal | Same as React |
| **Astro** | ~0KB (static) | Medium | Islands architecture |
| **Solid.js** | ~10KB | Medium | Fine-grained |
| **Svelte** | ~6.7KB | High (rewrite) | Compiler-based |
| **Vue** | ~34KB | High (rewrite) | Reactive refs |
| **Alpine.js** | ~15KB | High (different paradigm) | Declarative |
| **Vanilla JS** | 0KB | Medium | Manual DOM |

### React 19 (Current)
- **Pros:** Already in use, huge ecosystem, familiar
- **Cons:** Largest bundle (45KB), overkill for simple UI

### Preact
- **Pros:** 3KB, React-compatible API, same hooks (useState, useEffect, useRef)
- **Cons:** Smaller ecosystem, some React features missing (but not needed here)

### Astro
- **Pros:** Zero JS by default, islands architecture, framework-agnostic, excellent Cloudflare integration
- **Cons:** Overkill for pure SPAs, learning curve for islands mental model

### Solid.js
- **Pros:** Excellent performance, fine-grained reactivity
- **Cons:** Different mental model, requires learning new patterns, harder migration

### Svelte
- **Pros:** Compiler approach, no runtime, small output
- **Cons:** Complete rewrite required, different syntax, tooling complexity

### Alpine.js
- **Pros:** Progressive enhancement, simple syntax
- **Cons:** Different paradigm (HTML-first), not component-based

### Vanilla JS
- **Pros:** No framework overhead, full control
- **Cons:** Manual DOM manipulation, event listener management, more code

### Decision: Astro + Preact Islands

**Rationale:**
1. **Cloudflare acquisition (Jan 2026)** — Astro is now owned by Cloudflare, guaranteeing first-class integration with Workers, Pages, D1, KV, R2
2. **Zero JS for static pages** — Login, admin, docs pages ship no JavaScript
3. **Islands architecture** — Only hydrate interactive components (chat interface)
4. **Framework flexibility** — Use Preact for interactive islands, getting 3KB runtime
5. **Future-proof** — Easy to add landing pages, documentation, marketing content
6. **Excellent DX** — File-based routing, TypeScript support, built-in optimizations

**Architecture:**
```
Astro (static shell + routing)
  └── Preact islands (interactive components)
        ├── ChatInterface.tsx (client:load)
        ├── Stage1.tsx, Stage2.tsx, Stage3.tsx
        └── AdminPanel.tsx (client:load)
```

**Page-level breakdown:**
| Page | Rendering | JS Shipped |
|------|-----------|------------|
| `/` (landing) | Static | 0KB |
| `/login` | Static | ~1KB (form) |
| `/app` | Island | ~15KB (chat) |
| `/admin/flags` | Hybrid | ~5KB (toggle UI) |

**Current React features used:**
- `useState` ✓ (supported in Preact)
- `useEffect` ✓ (supported in Preact)
- `useRef` ✓ (supported in Preact)
- Context API ✗ (not used)
- Suspense ✗ (not used)
- useReducer ✗ (not used)

All current features are supported in Preact islands within Astro.

---

## Why Astro Over Next.js?

### What Next.js Provides
- Server-side rendering (SSR)
- Static site generation (SSG)
- File-based routing
- API routes
- Image optimization
- Built-in CSS/Sass support

### What Astro Provides
- Static-first with selective hydration (islands)
- Zero JS by default
- File-based routing
- Framework-agnostic (use any UI library)
- Built-in optimizations
- **Cloudflare ownership** (Jan 2026 acquisition)

### Comparison for LLM Council

| Feature | LLM Council Needs | Next.js | Astro |
|---------|-------------------|---------|-------|
| Static pages | Yes (login, admin) | Overkill | Perfect |
| Interactive chat | Yes | Full hydration | Island hydration |
| API routes | No (Hono backend) | Included | Not needed |
| Cloudflare integration | Critical | Good | First-class (owned) |
| Bundle size | <15KB target | ~80KB+ | ~0KB static, ~15KB islands |

### Why Astro Wins

1. **Cloudflare alignment** — Astro is now owned by Cloudflare. Integration with Workers, Pages, D1 will only improve.

2. **Right-sized hydration** — Next.js hydrates the entire page. Astro only hydrates interactive islands. For a chat app with static login/admin pages, this matters.

3. **No redundant features** — Next.js API routes are unused (we have Hono). Astro doesn't include them.

4. **Simpler mental model** — Static by default, opt-in to interactivity. Clear separation between content and application.

**Bottom line:** Astro + Preact islands gives us the best of both worlds — static pages ship zero JS, the chat interface ships minimal JS, and Cloudflare will prioritize Astro tooling going forward.

---

## Markdown Rendering Options

### Candidates

| Library | Size (gzip) | Features |
|---------|-------------|----------|
| **react-markdown** | ~12KB | Full CommonMark, plugins |
| **marked** | ~5KB | Fast, CommonMark |
| **markdown-it** | ~8KB | Extensible, plugins |
| **preact-markdown** | ~3KB | Minimal, Preact-native |

### Decision: marked + DOMPurify

**Rationale:**
1. Small combined size (~8KB)
2. Fast parsing
3. DOMPurify ensures XSS protection
4. More control over rendering
5. Well-maintained, widely used

**Implementation:**
```typescript
import { marked } from 'marked';
import DOMPurify from 'dompurify';

function Markdown({ children }: { children: string }) {
  const html = DOMPurify.sanitize(marked.parse(children));
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}
```

---

## Bundle Size Analysis

### Current (React)
```
react + react-dom:     ~45KB
react-markdown:        ~12KB
Application code:      ~10KB
─────────────────────────────
Total:                 ~67KB gzipped (all pages)
```

### Target (Astro + Preact Islands)

**Static pages (login, landing, docs):**
```
Astro runtime:         ~0KB (compiles to HTML)
Application code:      ~0KB (no JS needed)
─────────────────────────────
Total:                 ~0KB gzipped
```

**Interactive pages (chat app):**
```
preact:                ~3KB
marked:                ~5KB
dompurify:             ~3KB
Application code:      ~4KB (TypeScript compiled)
─────────────────────────────
Total:                 ~15KB gzipped
```

**Admin pages (feature flags):**
```
preact:                ~3KB (shared/cached)
Toggle components:     ~2KB
─────────────────────────────
Total:                 ~5KB gzipped
```

### Comparison

| Page Type | Current (React) | Target (Astro) | Reduction |
|-----------|-----------------|----------------|-----------|
| Login | 67KB | 0KB | 100% |
| Chat App | 67KB | 15KB | 78% |
| Admin | 67KB | 5KB | 93% |

**Average reduction: ~85% smaller bundles**

---

## SSE Streaming Implementation

### Python (FastAPI)
```python
from fastapi.responses import StreamingResponse

async def generate():
    yield f"data: {json.dumps({'type': 'stage1_start'})}\n\n"
    # ...

return StreamingResponse(generate(), media_type="text/event-stream")
```

### TypeScript (Hono)
```typescript
import { streamSSE } from 'hono/streaming';

app.post('/stream', (c) => {
  return streamSSE(c, async (stream) => {
    await stream.writeSSE({ data: JSON.stringify({ type: 'stage1_start' }) });
    // ...
  });
});
```

Both approaches are nearly identical in structure. Hono's `streamSSE` helper handles headers and formatting automatically.

---

## Async Parallelism

### Python (asyncio)
```python
tasks = [query_model(model, messages) for model in models]
results = await asyncio.gather(*tasks)
```

### TypeScript (Promise.all)
```typescript
const tasks = models.map(model => queryModel(model, messages));
const results = await Promise.all(tasks);
```

Direct 1:1 mapping. Both execute all requests concurrently.

---

## Summary of Decisions

| Layer | Choice | Key Reason |
|-------|--------|------------|
| Runtime | Bun | Single runtime, native TS, fast |
| Backend | Hono | Tiny, fast, portable |
| Frontend Shell | Astro | Zero JS static, Cloudflare-owned |
| Interactive UI | Preact (islands) | 3KB, React-compatible |
| Markdown | marked + DOMPurify | Small, secure |
| Types | Shared package | Single source of truth |
| Deployment | Cloudflare Pages | Native Astro support |

**Total expected bundle reduction:**
- Static pages: **~67KB → 0KB (100% smaller)**
- Interactive pages: **~67KB → ~15KB (78% smaller)**
- Average: **~85% smaller bundles**

---

## Astro + Cloudflare Integration

With Cloudflare's acquisition of Astro (January 2026), expect:

1. **Native adapter improvements** — `@astrojs/cloudflare` will become first-party
2. **D1 integration** — Direct database bindings in Astro components
3. **KV/R2 support** — Seamless asset and cache management
4. **Workers optimization** — Astro SSR optimized for Workers runtime
5. **Wrangler integration** — Single CLI for full-stack deployment

This acquisition de-risks the Astro choice significantly.
