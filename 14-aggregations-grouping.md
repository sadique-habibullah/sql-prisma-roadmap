# 14. Aggregations & Grouping

**Goal:** Summarize data with aggregate functions, group rows by a field, and filter groups with HAVING.
**Prerequisites:** [05 — SELECT Basics](./05-select-basics.md), [06 — Filtering with WHERE](./06-filtering-where.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Count all rows | `SELECT COUNT(*) FROM posts;` | `prisma.post.count()` |
| Count with filter | `SELECT COUNT(*) FROM posts WHERE published = true;` | `prisma.post.count({ where: { published: true } })` |
| Sum a column | `SELECT SUM(amount) FROM orders;` | `prisma.order.aggregate({ _sum: { amount: true } })` |
| Average | `SELECT AVG(price) FROM products;` | `prisma.product.aggregate({ _avg: { price: true } })` |
| Minimum | `SELECT MIN(price) FROM products;` | `prisma.product.aggregate({ _min: { price: true } })` |
| Maximum | `SELECT MAX(price) FROM products;` | `prisma.product.aggregate({ _max: { price: true } })` |
| Multiple aggregates together | `SELECT COUNT(*), AVG(price), MAX(price) FROM products;` | `prisma.product.aggregate({ _count: true, _avg: { price: true }, _max: { price: true } })` |
| GROUP BY | `SELECT author_id, COUNT(*) FROM posts GROUP BY author_id;` | `prisma.post.groupBy({ by: ['authorId'], _count: { id: true } })` |
| HAVING (filter groups) | `... HAVING COUNT(*) > 5` | `groupBy({ ..., having: { _count: { id: { gt: 5 } } } })` |
| COUNT DISTINCT | `SELECT COUNT(DISTINCT author_id) FROM posts;` | **No direct equivalent** — use `$queryRaw` |
| SUM with filter | `SELECT SUM(amount) FILTER (WHERE paid = true) FROM orders;` | **No direct equivalent** — use `$queryRaw` |
| Window functions | `ROW_NUMBER() OVER (PARTITION BY ...)` | **No Prisma equivalent** — use `$queryRaw` |

### Code: Basic aggregates

```sql
-- SQL: multiple aggregates in one query
SELECT
  COUNT(*)       AS total_posts,
  COUNT(*) FILTER (WHERE published = true) AS published_posts,
  MIN(created_at) AS oldest_post,
  MAX(created_at) AS newest_post
FROM posts;
```

```js
// Prisma: aggregate
const stats = await prisma.post.aggregate({
  _count: { id: true },
  _min:   { createdAt: true },
  _max:   { createdAt: true }
})
// stats._count.id, stats._min.createdAt, stats._max.createdAt
// Note: no FILTER equivalent — need $queryRaw for conditional aggregates
```

### Code: GROUP BY — posts per author

```sql
-- SQL
SELECT author_id, COUNT(*) AS post_count
FROM posts
GROUP BY author_id
ORDER BY post_count DESC;
```

```js
// Prisma
const results = await prisma.post.groupBy({
  by: ['authorId'],
  _count: { id: true },
  orderBy: {
    _count: { id: 'desc' }
  }
})
// results[0].authorId, results[0]._count.id
```

### Code: HAVING — only authors with more than 3 posts

```sql
-- SQL
SELECT author_id, COUNT(*) AS post_count
FROM posts
GROUP BY author_id
HAVING COUNT(*) > 3;
```

```js
// Prisma
const prolificAuthors = await prisma.post.groupBy({
  by: ['authorId'],
  _count: { id: true },
  having: {
    id: {
      _count: { gt: 3 }
    }
  }
})
```

### Code: Group with WHERE (filter before grouping)

```sql
-- SQL: only count published posts, grouped by author
SELECT author_id, COUNT(*) AS published_count
FROM posts
WHERE published = true
GROUP BY author_id;
```

```js
// Prisma
const results = await prisma.post.groupBy({
  by: ['authorId'],
  where: { published: true },
  _count: { id: true }
})
```

### Code: COUNT DISTINCT (via raw SQL)

```sql
-- SQL: count distinct authors who have published posts
SELECT COUNT(DISTINCT author_id) FROM posts WHERE published = true;
```

```js
// Prisma: no direct equivalent — fall back to $queryRaw
const result = await prisma.$queryRaw`
  SELECT COUNT(DISTINCT author_id)::int AS distinct_authors
  FROM posts
  WHERE published = true
`
// result[0].distinct_authors
```

---

## What's Really Happening

**`aggregate()` vs `groupBy()`** — `aggregate()` returns a single summary row across all matching records. `groupBy()` returns one row per unique combination of the `by` fields, each with its aggregates. They're equivalent to `SELECT COUNT(*) FROM posts` vs `SELECT author_id, COUNT(*) FROM posts GROUP BY author_id`.

**`_count: true` vs `_count: { fieldName: true }`** — in `aggregate()`, `_count: true` counts all rows. In `groupBy()`, `_count: { id: true }` counts non-null values of `id` within each group (analogous to `COUNT(id)` vs `COUNT(*)`).

**`having` syntax** — Prisma's `having` feels unintuitive at first. The structure is: `having: { fieldName: { _aggFunction: { comparisonOp: value } } }`. The `fieldName` must be in the `by` array or be aggregated.

**Window functions (`ROW_NUMBER`, `RANK`, `LAG`, `LEAD`, etc.)** — these are powerful SQL features with no Prisma equivalent. They're commonly used for rankings, running totals, and gap detection. Always use `$queryRaw` for these.

**`COUNT(*)` vs `COUNT(field)`** — `COUNT(*)` counts all rows including those with NULL values. `COUNT(field)` counts only non-null values of that field. Prisma's `_count: { fieldName: true }` is equivalent to `COUNT(fieldName)`.

> **Leaky abstraction alert:** Prisma's `groupBy` compiles to standard SQL `GROUP BY`, but the TypeScript types it returns may feel clunky — the aggregated result object nests under `_count`, `_sum`, etc. rather than column aliases. For complex reports or dashboards, raw SQL with typed results via `$queryRaw<ReportRow[]>` is often cleaner and more expressive.

---

## Practice

1. Write a SQL query and Prisma equivalent that counts the total number of posts, grouped by `published` (true/false).

2. Write a SQL query that finds all authors who have more than 2 published posts. Write the Prisma `groupBy` with a `having` clause equivalent.

3. Calculate the average body length of posts, grouped by author. Write the SQL first, then attempt the Prisma version — and note where you'd fall back to `$queryRaw`.

---

### Documentation Links

- PostgreSQL aggregate functions: [Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)
- PostgreSQL `GROUP BY` / `HAVING`: [Aggregate Queries](https://www.postgresql.org/docs/current/tutorial-agg.html)
- PostgreSQL window functions: [Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html)
- Prisma `aggregate`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#aggregate)
- Prisma `groupBy`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#groupby)
- Prisma aggregation docs: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/aggregation-grouping-summarizing)
