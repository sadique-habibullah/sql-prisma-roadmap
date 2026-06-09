# 18. Views

**Goal:** Create named virtual tables (views) that encapsulate reusable queries, and understand what Prisma can and cannot do with them.
**Prerequisites:** [05 — SELECT Basics](./05-select-basics.md), [13 — JOINs vs Includes](./13-joins-vs-includes.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Create a view | `CREATE VIEW published_posts AS SELECT ...` | `view PublishedPost { ... }` (Prisma 4.9+ with `views` preview) |
| Query a view | `SELECT * FROM published_posts;` | `prisma.publishedPost.findMany()` |
| Update a view | `CREATE OR REPLACE VIEW ...` | Update the `view` block + `prisma db pull` or `prisma migrate dev` |
| Drop a view | `DROP VIEW published_posts;` | Remove from schema + `prisma migrate dev` |
| Materialized view | `CREATE MATERIALIZED VIEW ...` | **No Prisma equivalent** — use `$queryRaw` to read, `$executeRaw` to refresh |
| Refresh materialized view | `REFRESH MATERIALIZED VIEW mv_stats;` | `prisma.$executeRaw\`REFRESH MATERIALIZED VIEW mv_stats\`` |
| Updatable view | A view without aggregates is updatable in Postgres | Prisma views are **read-only** — no `create`, `update`, `delete` |
| Security definer view | `CREATE VIEW ... WITH (security_invoker = true)` | Set in SQL; Prisma respects it when querying |
| View with `CHECK OPTION` | `CREATE VIEW ... WITH CHECK OPTION` | No Prisma equivalent for the definition |

### Code: Simple view

```sql
-- SQL: create a view for published posts with author name
CREATE VIEW published_posts AS
SELECT
  posts.id,
  posts.title,
  posts.created_at,
  users.name AS author_name
FROM posts
JOIN users ON posts.author_id = users.id
WHERE posts.published = true;

-- Query it
SELECT * FROM published_posts ORDER BY created_at DESC;
```

```prisma
// Prisma 4.9+ with preview feature
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["views"]
}

// After running prisma db pull, Prisma generates:
view PublishedPost {
  id         Int      @unique
  title      String
  createdAt  DateTime
  authorName String
}
```

```js
// Query the view just like a model
const posts = await prisma.publishedPost.findMany({
  orderBy: { createdAt: 'desc' }
})
```

### Code: Materialized view (no Prisma model support)

```sql
-- SQL: create a materialized view for post stats
CREATE MATERIALIZED VIEW post_stats AS
SELECT
  author_id,
  COUNT(*)                                    AS total_posts,
  COUNT(*) FILTER (WHERE published = true)    AS published_posts,
  MAX(created_at)                             AS last_posted_at
FROM posts
GROUP BY author_id;

-- Create an index on the materialized view
CREATE INDEX ON post_stats(author_id);

-- Refresh (re-run the query, update the stored data)
REFRESH MATERIALIZED VIEW post_stats;
```

```js
// Prisma: query via $queryRaw
const stats = await prisma.$queryRaw`
  SELECT * FROM post_stats WHERE author_id = ${userId}
`

// Refresh (e.g. on a schedule or after bulk inserts)
await prisma.$executeRaw`REFRESH MATERIALIZED VIEW post_stats`
```

### Code: Enabling Prisma views preview

```prisma
// schema.prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["views"]
}
```

```bash
# Pull existing views from the database into schema
npx prisma db pull

# Prisma will generate view blocks for any SQL views it finds
```

---

## What's Really Happening

**A view is a saved query** — it doesn't store data (unlike a table). Every time you `SELECT * FROM published_posts`, Postgres runs the underlying `SELECT ... FROM posts JOIN users ...` query. Views are useful for:
- Encapsulating complex JOINs so application code queries a simple name
- Restricting which columns or rows are visible (e.g. via Row Level Security)
- Providing a stable API to application code even if the underlying schema changes

**Materialized views store data** — unlike regular views, `CREATE MATERIALIZED VIEW` runs the query once and stores the result as if it were a table. Subsequent reads hit the stored copy, making reads much faster. The tradeoff: data becomes stale until you `REFRESH`. Use materialized views for expensive reporting queries where some staleness is acceptable.

**Prisma views are read-only** — Prisma's `view` block supports `findMany`, `findUnique`, `findFirst`, `count`, `aggregate`, and `groupBy`. Write operations (`create`, `update`, `delete`) are not supported on views. This matches PostgreSQL's behavior for views that aren't simple (e.g. views with JOINs or aggregations are not updatable anyway).

**Prisma views require `@unique` or `@@id`** — Prisma requires that a view block declares at least one field as `@unique` or `@@id` so it can identify rows. For views derived from a table with a primary key, include that column and mark it `@unique`.

**`db pull` discovers views** — if you create a view in SQL (e.g. in the Supabase SQL editor or a migration), running `npx prisma db pull` will add the `view` block to your schema. This is the typical workflow: write the SQL view in a migration, then pull the schema.

> **Leaky abstraction alert:** Prisma's `views` feature is still a preview feature as of Prisma 5.x. The API is stable for reads, but always check the Prisma changelog for updates before relying on it in production.

---

## Practice

1. Write a SQL `CREATE VIEW` called `active_users` that returns users who have at least one published post. Then write the Prisma `view` block for it and query it.

2. Create a materialized view `tag_usage` that counts how many posts use each tag. Write the SQL. Then write the Prisma `$queryRaw` to query it and `$executeRaw` to refresh it.

3. What is the key difference between a regular view and a materialized view? When would you choose each?

---

### Documentation Links

- PostgreSQL `CREATE VIEW`: [CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)
- PostgreSQL materialized views: [CREATE MATERIALIZED VIEW](https://www.postgresql.org/docs/current/sql-creatematerializedview.html)
- PostgreSQL `REFRESH MATERIALIZED VIEW`: [REFRESH MATERIALIZED VIEW](https://www.postgresql.org/docs/current/sql-refreshmaterializedview.html)
- Prisma views (preview): (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/views)
