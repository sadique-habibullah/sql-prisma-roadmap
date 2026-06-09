# 13. JOINs vs Includes

**Goal:** Understand how SQL JOINs map to Prisma `include`, and know the cases where Prisma's abstraction breaks down.
**Prerequisites:** [12 — Relationships](./12-relationships.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| INNER JOIN (fetch related rows) | `SELECT * FROM posts JOIN users ON posts.author_id = users.id` | `prisma.post.findMany({ include: { author: true } })` |
| LEFT JOIN (include even if no match) | `SELECT * FROM users LEFT JOIN posts ON users.id = posts.author_id` | `prisma.user.findMany({ include: { posts: true } })` — always includes (empty array if none) |
| Select specific columns on join | `SELECT p.title, u.name FROM posts p JOIN users u ON ...` | `include: { author: { select: { name: true } } }` |
| Filter on related table | `WHERE users.role = 'admin'` | `where: { author: { role: 'admin' } }` |
| Nested include (2 levels) | `JOIN users ON ... JOIN comments ON ...` | `include: { author: true, comments: { include: { author: true } } }` |
| Select with nested select | N/A (flat columns in SQL) | `select: { title: true, author: { select: { name: true } } }` |
| RIGHT JOIN | `RIGHT JOIN` | No direct equivalent — reverse the query |
| FULL OUTER JOIN | `FULL OUTER JOIN` | **No Prisma equivalent** — use `$queryRaw` |
| CROSS JOIN | `CROSS JOIN` | **No Prisma equivalent** — use `$queryRaw` |
| SELF JOIN | `FROM users u1 JOIN users u2 ON u1.manager_id = u2.id` | Use self-referential relation + `include` |
| COUNT on joined table | `COUNT(posts.id)` | `_count: { posts: true }` in `select` |

### Code: Fetch posts with their authors (INNER JOIN equivalent)

```sql
-- SQL
SELECT posts.*, users.name AS author_name, users.email AS author_email
FROM posts
JOIN users ON posts.author_id = users.id
WHERE posts.published = true;
```

```js
// Prisma — result is nested objects, not flat rows
const posts = await prisma.post.findMany({
  where: { published: true },
  include: { author: true }
})
// posts[0].author.name, posts[0].author.email
```

### Code: Select only specific nested fields

```sql
-- SQL
SELECT posts.id, posts.title, users.name
FROM posts
JOIN users ON posts.author_id = users.id;
```

```js
// Prisma
const posts = await prisma.post.findMany({
  select: {
    id: true,
    title: true,
    author: {
      select: { name: true }
    }
  }
})
// posts[0].author.name — other fields absent
```

### Code: Fetch users with their post count

```sql
-- SQL
SELECT users.*, COUNT(posts.id) AS post_count
FROM users
LEFT JOIN posts ON posts.author_id = users.id
GROUP BY users.id;
```

```js
// Prisma
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true,
    _count: {
      select: { posts: true }
    }
  }
})
// users[0]._count.posts === 3
```

### Code: Filter on related table (join condition in WHERE)

```sql
-- SQL: only posts written by admins
SELECT posts.*
FROM posts
JOIN users ON posts.author_id = users.id
WHERE users.role = 'admin';
```

```js
// Prisma
const posts = await prisma.post.findMany({
  where: {
    author: {
      role: 'admin'
    }
  }
})
```

### Code: Deep nested include

```sql
-- SQL: posts with author and all comments with their authors
SELECT p.*, u1.name AS author_name,
       c.body AS comment_body, u2.name AS commenter_name
FROM posts p
JOIN users u1 ON p.author_id = u1.id
LEFT JOIN comments c ON c.post_id = p.id
LEFT JOIN users u2 ON c.author_id = u2.id;
```

```js
// Prisma — cleanly nested
const posts = await prisma.post.findMany({
  include: {
    author: true,
    comments: {
      include: { author: true }
    }
  }
})
```

---

## What's Really Happening

**Prisma does not always generate a JOIN** — when you use `include`, Prisma may issue two separate SQL queries:
- Query 1: `SELECT * FROM posts WHERE ...`
- Query 2: `SELECT * FROM users WHERE id IN (1, 2, 3, ...)`

Then it merges the results in memory. This avoids the row-multiplication problem of JOINs on one-to-many relations (a user with 100 posts would appear 100 times in a JOIN). The exact strategy depends on the relation type and Prisma version.

**Result shape is different from SQL** — SQL returns flat rows with repeated data. Prisma returns a nested object graph. `posts[0].author.name` vs `row.author_name`. Both represent the same data; the shapes just differ. Adjust your mental model.

**LEFT JOIN is always the behavior for `include`** — Prisma's `include: { posts: true }` includes users *with no posts* (they'll have `posts: []`). It's always a LEFT JOIN semantics. If you want only users who have at least one post, add a `where` filter: `where: { posts: { some: {} } }`.

**RIGHT JOIN and FULL OUTER JOIN** have no Prisma equivalents. For a RIGHT JOIN, swap which model you're querying from. For FULL OUTER JOIN, use `$queryRaw`.

**N+1 problem** — if you load a list of posts and then loop over them calling `prisma.user.findUnique({ where: { id: post.authorId } })` for each, you're making N+1 queries (1 for posts + N for authors). Always use `include` or `select` to load related data in one query set.

> **Leaky abstraction alert:** Deep `include` chains on large datasets can return enormous response objects. A post with 1,000 comments each with their author is 1,001 records in memory. For large lists, prefer pagination + shallow `include`, or use `select` to limit fields aggressively.

---

## Practice

1. Write a SQL JOIN and a Prisma `include` that fetches all comments for a post with `id = 1`, along with each comment's author name.

2. Write a Prisma query that returns all users who have at least one post, with a count of their posts. Write the equivalent SQL.

3. A colleague wrote a loop that fetches posts and then queries each post's author individually. Rewrite it to use `include` and explain why it's faster.

---

### Documentation Links

- PostgreSQL `JOIN` types: [Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)
- Prisma `include`: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries#include-related-records)
- Prisma nested reads: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries#nested-reads)
- Prisma `select` on relations: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/select-fields#select-specific-relation-fields)
- Prisma `_count`: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/aggregation-grouping-summarizing#count-relations)
