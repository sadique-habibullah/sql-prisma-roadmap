# 19. Subqueries & CTEs

**Goal:** Write complex queries using subqueries and Common Table Expressions, and know when Prisma's relation filters cover the need vs. when to fall back to raw SQL.
**Prerequisites:** [06 — Filtering with WHERE](./06-filtering-where.md), [13 — JOINs vs Includes](./13-joins-vs-includes.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| IN subquery | `WHERE id IN (SELECT author_id FROM posts WHERE published = true)` | `where: { posts: { some: { published: true } } }` |
| NOT IN subquery | `WHERE id NOT IN (SELECT author_id FROM posts)` | `where: { posts: { none: {} } }` |
| EXISTS | `WHERE EXISTS (SELECT 1 FROM posts WHERE posts.author_id = users.id)` | `where: { posts: { some: {} } }` |
| NOT EXISTS | `WHERE NOT EXISTS (SELECT 1 FROM posts WHERE ...)` | `where: { posts: { none: {} } }` |
| ALL predicate | `WHERE id > ALL (SELECT ...)` | **No equivalent** — use `$queryRaw` |
| Scalar subquery | `SELECT (SELECT COUNT(*) FROM posts WHERE author_id = u.id) FROM users u` | **No equivalent** — use `$queryRaw` |
| Correlated subquery | Subquery references outer query's column | Prisma relation filters handle simple cases; complex cases need `$queryRaw` |
| Simple CTE | `WITH active AS (SELECT ...) SELECT * FROM active` | **No equivalent** — use `$queryRaw` |
| Multiple CTEs | `WITH a AS (...), b AS (...) SELECT ...` | **No equivalent** — use `$queryRaw` |
| Recursive CTE | `WITH RECURSIVE tree AS (...)` | **No equivalent** — use `$queryRaw` |
| CTE for `INSERT`/`UPDATE` | `WITH ... INSERT INTO ... SELECT ...` | **No equivalent** — use `$queryRaw` / `$executeRaw` |

### Code: IN subquery → Prisma relation filter

```sql
-- SQL: find users who have at least one published post
SELECT * FROM users
WHERE id IN (
  SELECT author_id FROM posts WHERE published = true
);
```

```js
// Prisma: relation filter with 'some'
const users = await prisma.user.findMany({
  where: {
    posts: { some: { published: true } }
  }
})
// Prisma compiles this to a subquery or EXISTS internally
```

### Code: EXISTS / NOT EXISTS

```sql
-- SQL: users with no posts
SELECT * FROM users
WHERE NOT EXISTS (
  SELECT 1 FROM posts WHERE posts.author_id = users.id
);
```

```js
// Prisma
const usersWithNoPosts = await prisma.user.findMany({
  where: {
    posts: { none: {} }
  }
})
```

### Code: Simple CTE → raw SQL

```sql
-- SQL: CTE to find the most-active author in the last 30 days
WITH recent_posts AS (
  SELECT author_id, COUNT(*) AS post_count
  FROM posts
  WHERE created_at > NOW() - INTERVAL '30 days'
  GROUP BY author_id
)
SELECT users.*, recent_posts.post_count
FROM users
JOIN recent_posts ON users.id = recent_posts.author_id
ORDER BY recent_posts.post_count DESC
LIMIT 1;
```

```js
// Prisma: no CTE support — use $queryRaw
const result = await prisma.$queryRaw`
  WITH recent_posts AS (
    SELECT author_id, COUNT(*) AS post_count
    FROM posts
    WHERE created_at > NOW() - INTERVAL '30 days'
    GROUP BY author_id
  )
  SELECT users.*, recent_posts.post_count
  FROM users
  JOIN recent_posts ON users.id = recent_posts.author_id
  ORDER BY recent_posts.post_count DESC
  LIMIT 1
`
```

### Code: Recursive CTE → raw SQL

```sql
-- SQL: walk a category tree (parent → children → grandchildren)
WITH RECURSIVE category_tree AS (
  -- Base case: root categories
  SELECT id, name, parent_id, 0 AS depth
  FROM categories
  WHERE parent_id IS NULL

  UNION ALL

  -- Recursive case: children
  SELECT c.id, c.name, c.parent_id, ct.depth + 1
  FROM categories c
  JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY depth, name;
```

```js
// Prisma: no recursive CTE support — must use $queryRaw
const tree = await prisma.$queryRaw`
  WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS depth
    FROM categories
    WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
  )
  SELECT * FROM category_tree ORDER BY depth, name
`
```

### Code: Relation filter operators (`some`, `none`, `every`)

```sql
-- SQL: users where ALL posts are published
SELECT * FROM users u
WHERE NOT EXISTS (
  SELECT 1 FROM posts p
  WHERE p.author_id = u.id AND p.published = false
);
```

```js
// Prisma: 'every' = all related records match the condition
const users = await prisma.user.findMany({
  where: {
    posts: { every: { published: true } }
  }
})
```

---

## What's Really Happening

**Prisma relation filters compile to subqueries** — `some`, `none`, `every`, and nested `where` on relations become `EXISTS` or `NOT EXISTS` subqueries (or JOINs) in the generated SQL. They handle the common case of "filter rows based on related rows" without requiring raw SQL.

| Prisma filter | SQL meaning |
|---|---|
| `posts: { some: { published: true } }` | `EXISTS (SELECT 1 FROM posts WHERE author_id = u.id AND published = true)` |
| `posts: { none: {} }` | `NOT EXISTS (SELECT 1 FROM posts WHERE author_id = u.id)` |
| `posts: { every: { published: true } }` | `NOT EXISTS (SELECT 1 FROM posts WHERE author_id = u.id AND published = false)` |

**CTEs are not supported in Prisma** — CTEs (`WITH ... AS (...)`) are one of the most powerful SQL features for breaking complex queries into named, readable steps. Prisma has no equivalent. Any time you need a CTE — for readability, performance, or recursive queries — reach for `$queryRaw`.

**Why CTEs matter** — beyond readability, CTEs can be used as optimization fences (Postgres materializes CTE results by default in older versions), for recursive tree walks, and for multi-step `INSERT`/`UPDATE` pipelines. None of these are expressible in Prisma's query API.

**Scalar subqueries** — `SELECT (SELECT COUNT(*) FROM posts WHERE author_id = u.id) FROM users` computes a per-row aggregate. Prisma has no equivalent. This is a common pattern for reports and dashboards — use `$queryRaw`.

> **Leaky abstraction alert:** Prisma's `every` filter has a subtle edge case: if a user has **no posts**, `every: { published: true }` returns `true` for that user (vacuous truth — all zero posts satisfy the condition). This matches SQL's `NOT EXISTS` behavior but can be surprising. If you want "users with at least one post where all posts are published", combine `some: {}` with `every: { published: true }`.

---

## Practice

1. Write a SQL `WHERE IN (subquery)` and the equivalent Prisma relation filter that finds all tags used on at least one published post.

2. Write a SQL recursive CTE that traverses a `categories` table (with `parent_id`) to find all descendants of category with `id = 1`. Write the `$queryRaw` equivalent.

3. Use Prisma to find users who have at least one post but all of their posts are unpublished. What filters do you combine? Write the SQL equivalent.

---

### Documentation Links

- PostgreSQL subqueries: [Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)
- PostgreSQL CTEs (`WITH`): [WITH Queries (Common Table Expressions)](https://www.postgresql.org/docs/current/queries-with.html)
- PostgreSQL recursive queries: [Recursive Queries](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE)
- Prisma relation filters: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries#relation-filters)
- Prisma raw queries: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access)
