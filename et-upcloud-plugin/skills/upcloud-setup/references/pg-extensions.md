# PostgreSQL Extensions Guide

Extensions enabled on the managed PostgreSQL instance to replace multiple separate services.

## pgvector — Vector Store

Replaces: Pinecone, Weaviate, Qdrant

```sql
CREATE EXTENSION IF NOT EXISTS vector;

-- Create a table with vector column
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content TEXT NOT NULL,
  embedding vector(1536),  -- OpenAI ada-002 dimension
  metadata JSONB DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Create HNSW index for fast similarity search
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- Similarity search
SELECT id, content, 1 - (embedding <=> $1::vector) AS similarity
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

## pg_cron — Job Scheduler

Replaces: External cron, SQS + Lambda, Cloud Scheduler

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Schedule a cleanup job (runs at midnight daily)
SELECT cron.schedule('cleanup-old-sessions', '0 0 * * *',
  $$DELETE FROM sessions WHERE expires_at < NOW()$$
);

-- Schedule analytics aggregation (every hour)
SELECT cron.schedule('hourly-stats', '0 * * * *',
  $$INSERT INTO hourly_stats (hour, count)
    SELECT date_trunc('hour', created_at), count(*)
    FROM events
    WHERE created_at > NOW() - interval '2 hours'
    GROUP BY 1
    ON CONFLICT (hour) DO UPDATE SET count = EXCLUDED.count$$
);

-- List scheduled jobs
SELECT * FROM cron.job;

-- Remove a job
SELECT cron.unschedule('cleanup-old-sessions');
```

## pg_trgm — Fuzzy Text Search

Replaces: Elasticsearch (for basic search), Algolia

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Create GIN index for trigram search
CREATE INDEX trgm_idx ON articles USING gin (title gin_trgm_ops);
CREATE INDEX trgm_content_idx ON articles USING gin (content gin_trgm_ops);

-- Fuzzy search
SELECT title, similarity(title, 'postgre') AS sim
FROM articles
WHERE title % 'postgre'  -- uses trigram similarity
ORDER BY sim DESC
LIMIT 10;

-- Combine with pgvector for hybrid search
SELECT a.title,
  (0.5 * similarity(a.title, $1) + 0.5 * (1 - (a.embedding <=> $2::vector))) AS score
FROM articles a
WHERE a.title % $1 OR a.embedding <=> $2::vector < 0.5
ORDER BY score DESC
LIMIT 10;
```

## JSONB — Document Store

Built-in, no extension needed. Replaces: MongoDB, Firestore

```sql
-- Flexible schema column
CREATE TABLE configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type TEXT NOT NULL,
  data JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- GIN index for fast JSONB queries
CREATE INDEX ON configs USING gin (data);

-- Query nested fields
SELECT * FROM configs WHERE data->>'status' = 'active';
SELECT * FROM configs WHERE data @> '{"tags": ["important"]}';

-- Partial index on JSONB
CREATE INDEX ON configs ((data->>'status')) WHERE data->>'status' IS NOT NULL;
```

## uuid-ossp — UUID Generation

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Use in default values
CREATE TABLE items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4()
);
```
