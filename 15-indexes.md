# 15. Indexes

**Goal:** Create indexes to speed up queries, understand index types, and know which index patterns Prisma can and cannot express.
**Prerequisites:** [04 — Tables & Models](./04-tables-and-models.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Simple index | `CREATE INDEX ON users(email);` | `@@index([email])` on the model |
| Named index | `CREATE INDEX idx_users_email ON users(email);` | `@@index([email], name: "idx_users_email")` |
| Composite index | `CREATE INDEX ON posts(author_id, created_at);` | `@@index([authorId, createdAt])` |
| Unique index (single) | `CREATE UNIQUE INDEX ON users(email);` | `email String @unique` |
| Unique index (composite) | `CREATE UNIQUE INDEX ON memberships(user_id, org_id);` | `@@unique([userId, orgId])` |
| Index on relation FK | `CREATE INDEX ON posts(author_id);` | Prisma does NOT auto-index FK columns — add `@@index([authorId])` manually |
| Partial index | `CREATE INDEX ON posts(created_at) WHERE published = true;` | **No Prisma equivalent** — add to migration SQL directly |
| GIN index (for JSONB/arrays) | `CREATE INDEX ON posts USING GIN(metadata);` | **No Prisma equivalent** — add to migration SQL directly |
| GiST index | `CREATE INDEX ON locations USING GIST(geom);` | **No Prisma equivalent** — add to migration SQL directly |
| BRIN index | `CREATE INDEX ON logs USING BRIN(created_at);` | **No Prisma equivalent** — add to migration SQL directly |
| Hash index | `CREATE INDEX ON users USING HASH(email);` | **No Prisma equivalent** |
| Drop index | `DROP INDEX idx_users_email;` | Remove from schema + `prisma migrate dev` |
| Index descending order | `CREATE INDEX ON posts(created_at DESC);` | `@@index([createdAt(sort: Desc)])` |

### Code: Adding indexes in Prisma schema

```prisma
model Post {
  id        Int      @id @default(autoincrement())
  title     String
  published Boolean  @default(false)
  authorId  Int
  createdAt DateTime @default(now())

  // Index the FK — Prisma does NOT do this automatically
  // Index on a single column
  // Composite index for common query pattern
  @@index([authorId])
  @@index([authorId, createdAt])
  @@index([createdAt(sort: Desc)])
}
```

### Code: Equivalent SQL

```sql
-- Prisma generates these in migrations:
CREATE INDEX "Post_authorId_idx"          ON posts("authorId");
CREATE INDEX "Post_authorId_createdAt_idx" ON posts("authorId", "createdAt");
CREATE INDEX "Post_createdAt_idx"          ON posts("createdAt" DESC);
```

### Code: Partial index (via raw migration SQL)

```sql
-- Prisma cannot generate this — add it manually in the migration file
-- or run it as $executeRaw in a seed/migration script
CREATE INDEX idx_posts_published ON posts(created_at)
WHERE published = true;
```

```js
// Via Prisma raw execution (e.g. in a seed or custom migration):
await prisma.$executeRaw`
  CREATE INDEX IF NOT EXISTS idx_posts_published
  ON posts(created_at)
  WHERE published = true
`
```

### Code: GIN index for JSONB (manual migration)

```sql
-- For querying JSONB metadata column efficiently
CREATE INDEX idx_posts_metadata_gin ON posts USING GIN(metadata);
```

```js
// Add in a custom migration or seed:
await prisma.$executeRaw`
  CREATE INDEX IF NOT EXISTS idx_posts_metadata_gin
  ON posts USING GIN(metadata)
`
```

---

## What's Really Happening

**What an index does** — an index is a separate data structure (usually a B-tree) that maps column values to row locations. Without an index, Postgres scans every row in the table (sequential scan). With an index, it jumps directly to matching rows (index scan). Indexes cost storage space and slow down writes slightly — don't index every column.

**Prisma does not auto-index FK columns** — this is a common gotcha. When you define `authorId Int` as a foreign key in Prisma, no index is created on that column. Every `WHERE authorId = 5` query will sequential-scan the `posts` table unless you explicitly add `@@index([authorId])`. Always add indexes on FK columns that you filter or sort by.

**When to add indexes:**
- Columns in `WHERE` clauses that are selective (high cardinality)
- FK columns you JOIN or filter on
- Columns used in `ORDER BY` when you also have a `WHERE`
- Columns used in `GROUP BY`

**Composite index column order matters** — an index on `(authorId, createdAt)` supports queries filtering on `authorId` alone, or on both `authorId` and `createdAt` together. It does NOT support queries filtering only on `createdAt`. Put the most selective or most-frequently-filtered column first.

**GIN indexes** are for JSONB columns and array columns. Standard B-tree indexes cannot index inside a JSONB blob. If you frequently query `WHERE metadata->>'key' = 'value'`, you need a GIN index.

**Partial indexes** are highly efficient for queries on a subset of rows. `CREATE INDEX ON posts(created_at) WHERE published = true` is smaller and faster than indexing all posts — it only covers the rows you actually query.

> **Leaky abstraction alert:** `@unique` in Prisma creates both a `UNIQUE` constraint AND a unique index — they're the same object in PostgreSQL. `@@index` creates a non-unique index. Use `@unique`/`@@unique` when you need uniqueness guarantees; use `@@index` when you just want query performance.

---

## Practice

1. Look at the unified schema from [04 — Tables & Models](./04-tables-and-models.md). Which FK columns need explicit `@@index` annotations? Add them and write the SQL equivalents.

2. Write a partial index (SQL) for the `posts` table that indexes `created_at` only for unpublished posts. Why is this more efficient than a full index?

3. You frequently run `SELECT * FROM posts WHERE author_id = ? ORDER BY created_at DESC`. What composite index would you create? Write both the SQL and the Prisma `@@index`.

---

### Documentation Links

- PostgreSQL `CREATE INDEX`: [CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html)
- PostgreSQL index types (B-tree, GIN, GiST, etc.): [Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- PostgreSQL partial indexes: [Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)
- Prisma `@@index`: (verify: https://www.prisma.io/docs/orm/reference/prisma-schema-reference#index)
- Prisma indexes overview: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes)
