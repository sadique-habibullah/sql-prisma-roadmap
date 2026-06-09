# 23. Performance & Connection Pooling

**Goal:** Diagnose slow queries with EXPLAIN/ANALYZE, understand common performance pitfalls, and configure Supabase connection pooling correctly for Prisma.
**Prerequisites:** [15 — Indexes](./15-indexes.md), [01 — Setup & Environment](./01-setup-and-environment.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Explain a query | `EXPLAIN SELECT * FROM users WHERE email = 'a@b.com';` | `prisma.$queryRaw\`EXPLAIN SELECT ...\`` |
| Explain + run (actual times) | `EXPLAIN ANALYZE SELECT ...` | `prisma.$queryRaw\`EXPLAIN ANALYZE SELECT ...\`` |
| Sequential scan | `Seq Scan on users` in EXPLAIN output | Indicates missing index — add `@@index` |
| Index scan | `Index Scan using users_email_idx on users` | Expected output with correct index |
| Bitmap heap scan | `Bitmap Heap Scan` in EXPLAIN output | Index used, but fetching many rows |
| N+1 queries | Loop with individual SELECT per row | Use `include` or `select` instead of nested loops |
| Supavisor (pooler) | Port 6543, transaction mode by default | `DATABASE_URL=...?pgbouncer=true` |
| Supavisor session mode | Port 5432 (direct) or configured port | Use `DIRECT_URL` or configure Supabase to use session mode |
| `pgbouncer=true` flag | Disables prepared statements in PgBouncer | Add to `DATABASE_URL` query string — Prisma uses simple query protocol |
| Max connections | `SHOW max_connections;` | Supabase free tier: ~60 direct connections; pooler multiplexes thousands |
| Check active connections | `SELECT count(*) FROM pg_stat_activity;` | `prisma.$queryRaw\`SELECT count(*) FROM pg_stat_activity\`` |
| Connection timeout | Connection pool setting | `prisma.$connect()` fails if DB unreachable |
| Slow query log | `log_min_duration_statement = 1000` in `postgresql.conf` | Enable in Supabase dashboard under Database Settings |

### Code: Running EXPLAIN via Prisma

```js
// Analyze what PostgreSQL does for a query
const plan = await prisma.$queryRaw`
  EXPLAIN ANALYZE
  SELECT * FROM posts
  WHERE author_id = 5
  ORDER BY created_at DESC
  LIMIT 10
`
console.log(plan)
// Look for: "Seq Scan" (bad) vs "Index Scan" (good)
// Look for: actual time, rows, loops
```

### Code: Reading EXPLAIN output

```
-- With no index on author_id:
Seq Scan on posts  (cost=0.00..45.00 rows=3 width=100) (actual time=0.02..5.43 rows=3 loops=1)
  Filter: (author_id = 5)
  Rows Removed by Filter: 997

-- After adding @@index([authorId]):
Index Scan using "Post_authorId_idx" on posts  (cost=0.28..12.30 rows=3 width=100) (actual time=0.05..0.08 rows=3 loops=1)
  Index Cond: (author_id = 5)
```

### Code: Diagnosing the N+1 problem

```js
// BAD: N+1 — 1 query for posts + N queries for each author
const posts = await prisma.post.findMany()
for (const post of posts) {
  const author = await prisma.user.findUnique({ where: { id: post.authorId } })
  // This fires a separate query for EACH post!
}

// GOOD: 1 or 2 queries total
const posts = await prisma.post.findMany({
  include: { author: true }  // Prisma batches author queries
})
```

### Code: Supabase connection pooling setup

```env
# DATABASE_URL: Supavisor (transaction mode, port 6543)
# ?pgbouncer=true disables prepared statements (required for transaction mode)
# connection_limit=1 prevents each serverless function from opening many connections
DATABASE_URL="postgresql://postgres.xxxx:password@aws-0-us-east-1.pooler.supabase.com:6543/postgres?pgbouncer=true&connection_limit=1"

# DIRECT_URL: Direct connection (port 5432) for migrations
DIRECT_URL="postgresql://postgres:password@db.xxxx.supabase.co:5432/postgres"
```

### Code: Check current connections

```js
// How many connections are open right now?
const result = await prisma.$queryRaw`
  SELECT
    count(*) AS total,
    count(*) FILTER (WHERE state = 'active') AS active,
    count(*) FILTER (WHERE state = 'idle')   AS idle
  FROM pg_stat_activity
  WHERE datname = current_database()
`
console.log(result[0])
```

---

## What's Really Happening

**Supabase's connection architecture:**

```
Your App (many instances)
    │
    ▼
Supavisor (port 6543)    ← connection pooler
    │  multiplexes N app connections onto M DB connections
    ▼
PostgreSQL (port 5432)   ← actual database
```

Supabase's free tier allows ~60 direct database connections. Your application might have dozens of serverless function instances, each wanting a connection. Without a pooler, you'd hit that limit immediately. Supavisor allows hundreds of app connections while keeping only a few real Postgres connections open.

**Transaction mode vs session mode:**
- **Transaction mode** (default at port 6543): the pooler assigns a Postgres connection for the duration of a single transaction, then returns it to the pool. This is efficient but means you can't use session-level features (prepared statements, temporary tables, advisory locks that span transactions).
- **Session mode**: each app connection gets its own Postgres connection for its lifetime. More connections held, but session-level features work.

**`pgbouncer=true` disables prepared statements** — in transaction mode, a prepared statement from connection A is unavailable to connection B (different Postgres backend). `pgbouncer=true` tells Prisma to use the "simple query" protocol instead of the "extended query" protocol (which uses prepared statements). With this flag, each query is sent as a plain text SQL string — slightly less efficient per query, but works correctly with the pooler.

**`connection_limit=1`** — in serverless environments (Vercel, Netlify Functions, AWS Lambda), each function invocation creates a new `PrismaClient` instance. Without limiting connections, a traffic spike creates hundreds of connections simultaneously. `connection_limit=1` ensures each `PrismaClient` only opens one connection; Supavisor handles multiplexing.

**Identifying slow queries:**
1. Use `EXPLAIN ANALYZE` to see the actual execution plan.
2. Look for `Seq Scan` on large tables — they indicate a missing index.
3. Look at `actual time` in the output — high times relative to `cost` estimates indicate stale statistics (run `ANALYZE table_name`).
4. Enable Supabase's slow query log in the dashboard (Database Settings → Logs).

> **Leaky abstraction alert:** Prisma's connection pool operates at the client level. By default (not serverless), `PrismaClient` maintains a pool of several connections. In serverless, you want `connection_limit=1` to prevent exhausting the Postgres connection limit. Always check the Supabase dashboard's "Database → Connections" view to see how many connections your app is actually using.

---

## Practice

1. Add a `views Int @default(0)` column to `Post`. Insert 1,000 posts. Run `EXPLAIN ANALYZE SELECT * FROM posts WHERE author_id = 1 ORDER BY views DESC LIMIT 10`. Read the output — is an index being used?

2. Identify the N+1 problem in this code and fix it:
   ```js
   const posts = await prisma.post.findMany({ where: { published: true } })
   const enriched = await Promise.all(posts.map(p =>
     prisma.user.findUnique({ where: { id: p.authorId } }).then(a => ({ ...p, author: a }))
   ))
   ```

3. Your Supabase project is on the free tier (60 connections). You're deploying to Vercel (up to 10 concurrent function instances). How should you configure `DATABASE_URL` to avoid hitting the connection limit?

---

### Documentation Links

- PostgreSQL `EXPLAIN`: [EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)
- PostgreSQL `EXPLAIN ANALYZE`: [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- PostgreSQL index usage: [Indexes](https://www.postgresql.org/docs/current/indexes.html)
- Supabase connection pooling (Supavisor): (verify: https://supabase.com/docs/guides/database/connecting-to-postgres#connection-pooler)
- Supabase connection management: (verify: https://supabase.com/docs/guides/database/connecting-to-postgres)
- Prisma connection management: (verify: https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections)
- Prisma serverless best practices: (verify: https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/serverless-environments)
