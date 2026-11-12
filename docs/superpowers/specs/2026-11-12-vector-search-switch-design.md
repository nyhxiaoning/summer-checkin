# Vector Search Switch Design

## Goal

Add one server-side environment switch, `VECTOR_SEARCH_ENABLED`, that disables vector-dependent runtime behavior. When set to `false`, the application must not call an embedding API and must not execute PostgreSQL `vector` casts or distance operators.

The disabled mode must remain useful:

- Knowledge documents are still stored and split into chunks.
- Knowledge retrieval falls back to PostgreSQL keyword matching.
- Long-term memory is still stored.
- Memory retrieval falls back to importance and recency.
- Authentication, plans, check-ins, documents, chat, and other non-vector features continue to work.

## Configuration Contract

Create a small server-only configuration module exposing:

```ts
export const vectorSearchEnabled: boolean;
```

Parsing rules:

- Missing `VECTOR_SEARCH_ENABLED` means `true`, preserving current production behavior.
- The exact case-insensitive string `false` means disabled.
- Any other value means enabled.

Document the variable in `.env.example` and pass it into the application container from `docker-compose.yml`:

```dotenv
VECTOR_SEARCH_ENABLED=false
```

No `NEXT_PUBLIC_` prefix is used because this is a server-only capability switch.

## Approaches Considered

### 1. Disable affected endpoints entirely

This is the smallest code change, but it would make knowledge upload, RAG, and parts of long-term memory unavailable. It does not meet the requirement that the application remain useful when vector behavior is disabled.

### 2. JSON embeddings plus application-side cosine similarity

This preserves semantic search but still calls embedding APIs, loads vectors into Node.js, and recreates the old scalability and truncation problems. It also requires a database representation migration. This is rejected.

### 3. Capability switch with deterministic fallbacks

This is the selected approach. Enabled mode remains unchanged. Disabled mode avoids embedding generation and vector SQL, stores ordinary relational data, uses keyword retrieval for knowledge, and importance/recency retrieval for memory.

## Schema Strategy

Keep all three fields as `Unsupported("vector(1024)")` so enabled deployments and existing production data retain their current database representation.

Change only `DocumentChunk.embedding` from required to optional:

```prisma
embedding Unsupported("vector(1024)")?
```

`UserMemory.embedding` and `Memory.embedding` are already optional. Disabled-mode knowledge chunks can therefore be stored with `embedding = null`.

This switch is deliberately a runtime capability switch, not a database-type migration. PostgreSQL must still have the pgvector extension installed before `prisma db push`, because the schema continues to contain vector columns. Supporting a database with no pgvector extension at all requires a separate storage redesign and migration; mixing that migration into this switch would risk converting or dropping production vector data.

## Knowledge Upload Flow

### Enabled

Preserve the current flow:

1. Split source text.
2. Generate embeddings outside the transaction.
3. Store chunks and embeddings atomically.
4. Use vector similarity for retrieval.

### Disabled

Use this flow:

1. Split source text.
2. Do not call `embedTexts`.
3. Store chunks with `embedding = null` in the same replacement transaction.
4. Store the source document as today.

The upload result and replacement guarantees remain unchanged.

## Knowledge Retrieval Flow

The public search interface remains unchanged.

### Enabled

Generate a query embedding and execute the current pgvector search.

### Disabled

Do not generate a query embedding. Search chunks using PostgreSQL text matching scoped by `userId` and optional `sourceName`.

Ranking rules:

1. Case-insensitive exact phrase matches first.
2. Remaining case-insensitive substring matches next.
3. Newer chunks break ties.
4. Respect the existing `topK` limit.

The fallback returns a deterministic synthetic similarity score so downstream result shapes do not change. Exact phrase matches use `1`; substring matches use `0.5`.

## Long-Term Memory Write Flow

### Enabled

Preserve current embedding generation and vector storage.

### Disabled

- Do not call the embedding provider.
- Create memory rows with `embedding = null` and `embeddingModel = null`.
- `setMemoryEmbedding` becomes a no-op returning zero affected rows.
- The lifecycle job does not attempt to backfill missing embeddings.

## Long-Term Memory Retrieval Flow

### Enabled

Preserve vector similarity search.

### Disabled

- Duplicate detection by vector returns no match; normalized-key logic remains active.
- Extraction context uses the existing importance-descending and updated-time-descending fallback.
- Query-related recall returns active memories ordered by importance and recency, subject to the same result limit.
- Tool-driven memory search uses the same fallback and does not call the embedding provider.

## Error Handling

- Disabled mode must never fail because an embedding API key is absent.
- Disabled mode must never issue SQL containing `::vector`, `<=>`, `vector_dims`, or HNSW operations.
- Enabled mode preserves current errors and behavior.
- Invalid switch values do not crash startup; only `false` disables the feature.

## Tests

Add focused tests for:

1. Environment parsing: missing, `true`, `false`, and mixed-case `FALSE`.
2. Knowledge upload in disabled mode does not call the embedding client and writes null embeddings.
3. Knowledge retrieval in disabled mode issues keyword SQL and no vector SQL.
4. Memory creation in disabled mode does not call the embedding provider.
5. Memory retrieval in disabled mode uses importance/recency and no vector SQL.
6. Existing enabled-mode tests continue to pass.

Run:

```bash
npm test
npm run typecheck
npm run lint
```

## Deployment

### pgvector installation prerequisite

`VECTOR_SEARCH_ENABLED=false` disables vector behavior at runtime, but the current Prisma schema still declares `Unsupported("vector(1024)")`. The PostgreSQL server must therefore have the pgvector extension files installed, and the extension must be enabled in `summer_checkin` before `prisma db push`.

The recommended development setup uses the repository's PostgreSQL 16 image, which already contains the extension files:

```bash
# Ensure native PostgreSQL is not already occupying port 5432, then start the container
docker compose up -d postgres

# Enable pgvector once in the application database
docker exec postgresServer \
  psql -U postgres -d summer_checkin \
  -c 'CREATE EXTENSION IF NOT EXISTS vector;'

# Verify installation
docker exec postgresServer \
  psql -U postgres -d summer_checkin \
  -c "SELECT extversion FROM pg_extension WHERE extname = 'vector';"

# Create the Prisma-managed tables and seed data
npx prisma generate
npx prisma db push
npx prisma db seed
```

The current native local server is Homebrew PostgreSQL 14. The current Homebrew `pgvector` formula only installs into supported newer Homebrew PostgreSQL formulas, so `brew install pgvector` is not a reliable installation path for this PostgreSQL 14 instance. Prefer the Docker PostgreSQL 16 setup above.

If PostgreSQL 14 must be retained, compile pgvector against that exact server's `pg_config`:

```bash
git clone --branch v0.8.7 https://github.com/pgvector/pgvector.git
cd pgvector
export PG_CONFIG="$(brew --prefix postgresql@14)/bin/pg_config"
make clean
make
make install
```

Then create or select `summer_checkin` with an administrative role and enable the extension:

```bash
psql -U henryheng -d summer_checkin \
  -c 'CREATE EXTENSION IF NOT EXISTS vector;'
```

Installing files and enabling the extension are separate operations. `CREATE EXTENSION vector` must be executed once in every database that uses vector columns.

Development with vector runtime behavior disabled:

```dotenv
VECTOR_SEARCH_ENABLED=false
```

Production with pgvector:

```dotenv
VECTOR_SEARCH_ENABLED=true
```

Docker Compose passes the value through with an enabled default so existing deployments do not silently lose semantic retrieval:

```yaml
VECTOR_SEARCH_ENABLED: ${VECTOR_SEARCH_ENABLED:-true}
```

## Success Criteria

- The server starts without an embedding API key in disabled mode.
- Knowledge upload and keyword retrieval work in disabled mode.
- Long-term memory storage and non-vector recall work in disabled mode.
- No vector SQL executes in disabled mode.
- Existing pgvector behavior is unchanged when the switch is enabled.
- Prisma schema and existing production vector column types remain unchanged, except that `DocumentChunk.embedding` becomes nullable.

