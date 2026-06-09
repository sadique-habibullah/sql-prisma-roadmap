# 10. Sorting & Pagination

**Goal:** Control result ordering and page through large datasets efficiently — including cursor-based pagination.
**Prerequisites:** [06 — Filtering with WHERE](./06-filtering-where.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Sort ascending | `ORDER BY name ASC` | `orderBy: { name: 'asc' }` |
| Sort descending | `ORDER BY created_at DESC` | `orderBy: { createdAt: 'desc' }` |
| Sort by multiple columns | `ORDER BY role ASC, created_at DESC` | `orderBy: [{ role: 'asc' }, { createdAt: 'desc' }]` |
| Limit results | `LIMIT 10` | `take: 10` |
| Skip rows (offset) | `OFFSET 20` | `skip: 20` |
| Page-based pagination | `LIMIT 10 OFFSET 20` | `take: 10, skip: 20` |
| Cursor-based pagination | `WHERE id > :last_id ORDER BY id ASC LIMIT 10` | `cursor: { id: lastId }, take: 10, skip: 1, orderBy: { id: 'asc' }` |
| NULLS LAST | `ORDER BY name ASC NULLS LAST` | `orderBy: { name: { sort: 'asc', nulls: 'last' } }` |
| NULLS FIRST | `ORDER BY name DESC NULLS FIRST` | `orderBy: { name: { sort: 'desc', nulls: 'first' } }` |
| Random order | `ORDER BY RANDOM()` | No equivalent — use `$queryRaw` |

### Code: Sort and limit

```sql
-- SQL: 10 most recent posts
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 10;
```

```js
// Prisma
const posts = await prisma.post.findMany({
  orderBy: { createdAt: 'desc' },
  take: 10
})
```

### Code: Offset pagination (page 3, 10 per page)

```sql
-- SQL: page 3 (items 21–30)
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 10 OFFSET 20;
```

```js
// Prisma
const page = 3
const pageSize = 10

const posts = await prisma.post.findMany({
  orderBy: { createdAt: 'desc' },
  take: pageSize,
  skip: (page - 1) * pageSize  // skip = 20
})
```

### Code: Cursor pagination (after a known ID)

```sql
-- SQL: next 10 posts after post with id=42
SELECT * FROM posts
WHERE id > 42
ORDER BY id ASC
LIMIT 10;
```

```js
// Prisma
const posts = await prisma.post.findMany({
  orderBy: { id: 'asc' },
  cursor: { id: 42 },  // start FROM this record...
  skip: 1,             // ...but skip the cursor record itself
  take: 10
})
```

### Code: Multi-column sort

```sql
-- SQL: sort by role ascending, then by name descending within each role
SELECT * FROM users
ORDER BY role ASC, name DESC;
```

```js
// Prisma
const users = await prisma.user.findMany({
  orderBy: [
    { role: 'asc' },
    { name: 'desc' }
  ]
})
```

### Code: NULLS LAST

```sql
-- SQL: users with a bio first, NULL bios last
SELECT * FROM users ORDER BY bio ASC NULLS LAST;
```

```js
// Prisma
const users = await prisma.user.findMany({
  orderBy: {
    bio: { sort: 'asc', nulls: 'last' }
  }
})
```

---

## What's Really Happening

**Offset pagination performance** — `LIMIT 10 OFFSET 10000` is slow because Postgres must scan and discard the first 10,000 rows before returning the 10 you want. At high page numbers on large tables, this becomes a full table scan. Offset pagination is acceptable for small datasets or admin UIs, but it's a performance problem at scale.

**Cursor pagination** — instead of skipping rows, cursor pagination uses a `WHERE id > :last_seen_id` condition. With an index on `id`, Postgres jumps directly to the right position — O(log n) regardless of page depth. The tradeoff: you can't jump to page 50 directly; you can only go forward (or backward with careful design).

**Prisma cursor + `skip: 1`** — Prisma's cursor is *inclusive*: the cursor record itself is included in results. Adding `skip: 1` skips past the cursor record so you don't see it again. This is the standard pattern.

**`take` as a negative number** — Prisma supports `take: -10` to fetch the 10 rows *before* the cursor (going backward). Pair with `cursor` and `skip: 1` to implement a "previous page" link.

**`ORDER BY RANDOM()`** — there is no Prisma equivalent. For random sampling, use `prisma.$queryRaw\`SELECT * FROM users ORDER BY RANDOM() LIMIT 5\``. At large table sizes, `ORDER BY RANDOM()` is also slow — it sorts the entire result set. Use `TABLESAMPLE` for large-scale random sampling.

> **Leaky abstraction alert:** Prisma's cursor pagination only works correctly when the cursor field has a unique index (like `@id`). Using a non-unique cursor field (like `createdAt`) can cause records to be skipped or repeated when multiple rows share the same cursor value.

---

## Practice

1. Write a SQL query and Prisma call that returns page 4 of posts (10 per page), sorted by `createdAt` descending.

2. Implement cursor-based pagination: given the `id` of the last post seen (`lastId = 55`), write both SQL and Prisma to fetch the next 5 published posts sorted by `id` ascending.

3. Write a SQL query that sorts users by `role` ascending and by `name` descending within each role, with NULL names sorted last. Then write the Prisma equivalent.

---

### Documentation Links

- PostgreSQL `ORDER BY` / `LIMIT` / `OFFSET`: [SELECT — Sorting Rows](https://www.postgresql.org/docs/current/queries-order.html)
- PostgreSQL `NULLS FIRST/LAST`: [Sorting Rows](https://www.postgresql.org/docs/current/queries-order.html)
- Prisma `orderBy`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#orderby)
- Prisma pagination: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/pagination)
- Prisma cursor-based pagination: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/pagination#cursor-based-pagination)
