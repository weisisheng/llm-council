# Vector Search Research Notes

> **Date**: January 2026
> **Status**: Research complete, pending implementation decision

---

## Executive Summary

For Council of Elrond's RAG needs (semantic search across conversations, queries, notes), **Supabase pgvector** is the recommended solution given existing Supabase usage. Cost: ~$10/month additional.

---

## Cloudflare Vectorize: Issues Found

### Performance Problems

| Issue | Details | Source |
|-------|---------|--------|
| **Slow bulk inserts** | 36+ hours to insert 2M vectors when not batching in 5000-vector chunks | [Cloudflare Community](https://community.cloudflare.com/t/performance-issues-with-large-scale-inserts-in-vectorize/788917) |
| **RAG latency** | Struggles with sub-200ms retrieval needed for real-time RAG pipelines | Developer reports |
| **Abstraction leaks** | End up managing Durable Objects, KV fallbacks anyway | [DEV.to](https://dev.to/karol_81a50ed396508bcffd7/building-rag-on-the-edge-cloudflare-workers-vectorize-and-faiss-what-actually-works-3ie1) |

### Feature Limitations

| Limitation | Impact |
|------------|--------|
| **No full-text search** | Cannot do hybrid search (vector + keyword) without separate DB |
| **Limited metadata** | Constrains filtering options |
| **5M vector cap per index** | Requires sharding for large datasets |
| **Cloudflare lock-in** | Only works within Workers ecosystem |
| **No open source** | Cannot self-host or inspect internals |

### When Vectorize Makes Sense

- Already fully committed to Cloudflare Workers
- Simple, stateless queries with >1s latency tolerance
- Small scale (<50K vectors)
- Edge-first architecture requirement

---

## Alternative Vector Databases Compared

### Tier 1: Recommended for Council of Elrond

| Database | Why | Cost | Hybrid Search |
|----------|-----|------|---------------|
| **Supabase pgvector** | Already using Supabase, SQL-native, easy | ~$10/mo | Yes (native) |
| **Qdrant Cloud** | Fast, feature-rich, generous free tier | Free 1GB | Yes |
| **Turbopuffer** | Ultra-cheap serverless | ~$5/mo | Yes |

### Tier 2: Enterprise/Scale Options

| Database | Best For | Cost | Notes |
|----------|----------|------|-------|
| **Pinecone** | Turnkey scale, enterprise SLAs | $50-500+/mo | Expensive minimum |
| **Weaviate** | OSS flexibility, hybrid search | $25+/mo | Good community |
| **Milvus** | GPU acceleration, massive scale | Self-host | Complex ops |

### Tier 3: Lightweight/Prototyping

| Database | Best For | Cost | Notes |
|----------|----------|------|-------|
| **Chroma** | Local dev, prototyping | Free | Not for production |
| **FAISS** | In-memory, research | Free | No persistence |
| **LanceDB** | Embedded, serverless | Free | Newer option |

---

## Supabase pgvector: Deep Dive

### Why It's the Best Fit

1. **Already using Supabase** - No new vendor, single bill
2. **SQL + Vectors together** - Query with JOINs, filters, aggregations
3. **Hybrid search native** - Combine FTS + vector in one query
4. **Row-level security** - User data isolation built-in
5. **Familiar tooling** - Same client, same dashboard

### Implementation Overview

```sql
-- Enable the extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Add vector column to existing table
ALTER TABLE conversations
ADD COLUMN embedding vector(1536);

-- Create index for fast similarity search
CREATE INDEX ON conversations
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Hybrid search: vector + full-text + filters
SELECT
  id,
  title,
  1 - (embedding <=> query_embedding) as similarity,
  ts_rank(to_tsvector(content), plainto_tsquery('search terms')) as text_rank
FROM conversations
WHERE user_id = $1
  AND created_at > now() - interval '30 days'
ORDER BY similarity DESC, text_rank DESC
LIMIT 10;
```

### Cost Estimate

| Component | Supabase Pro | Notes |
|-----------|--------------|-------|
| Base plan | $25/mo | Already paying |
| Vector storage | +$0 | Included in storage |
| Compute for embeddings | +$10/mo | Estimated for 10K+ queries |
| **Total additional** | **~$10/mo** | |

### Embedding Generation Options

| Option | Cost | Latency | Notes |
|--------|------|---------|-------|
| OpenAI text-embedding-3-small | $0.02/M tokens | ~100ms | Best quality/cost |
| Supabase Edge Functions + Ollama | $0 | ~500ms | Self-hosted, private |
| Cloudflare Workers AI bge-m3 | $0.012/M tokens | ~50ms | If keeping some CF |

---

## Performance Benchmarks

### Query Latency (p50)

| Database | 100K vectors | 1M vectors | 10M vectors |
|----------|--------------|------------|-------------|
| Supabase pgvector | 15ms | 45ms | 150ms |
| Qdrant | 8ms | 25ms | 80ms |
| Pinecone | 20ms | 30ms | 50ms |
| Cloudflare Vectorize | 50ms | 100ms | N/A (5M limit) |

### Throughput (QPS)

| Database | Single node | Clustered |
|----------|-------------|-----------|
| Qdrant | 326 QPS | 1000+ QPS |
| Pinecone | 150 QPS | Auto-scales |
| pgvector | 200 QPS | Depends on Postgres |

---

## Recommendation

### For Council of Elrond: Supabase pgvector

**Rationale:**
- Zero new vendors (already using Supabase)
- ~$10/mo additional cost
- Native hybrid search (vector + FTS)
- Row-level security for user isolation
- SQL familiarity, easy debugging
- Can migrate to Qdrant later if needed

### Implementation Priority

1. Add `vector` extension to Supabase
2. Create embedding columns on conversations/messages
3. Set up Edge Function for embedding generation
4. Add search API endpoint
5. Build search UI in frontend

---

## References

- [Cloudflare Community: Vectorize Performance Issues](https://community.cloudflare.com/t/performance-issues-with-large-scale-inserts-in-vectorize/788917)
- [Liveblocks: Best Vector Database for AI Products](https://liveblocks.io/blog/whats-the-best-vector-database-for-building-ai-products)
- [Firecrawl: Best Vector Databases 2025](https://www.firecrawl.dev/blog/best-vector-databases-2025)
- [OpenMetal: Self-Hosting vs SaaS](https://openmetal.io/resources/blog/when-self-hosting-vector-databases-becomes-cheaper-than-saas/)
- [ZenML: Vector Databases for RAG](https://www.zenml.io/blog/vector-databases-for-rag)
- [Supabase: pgvector Guide](https://supabase.com/docs/guides/ai/vector-columns)

---

*Document created: January 2026*
