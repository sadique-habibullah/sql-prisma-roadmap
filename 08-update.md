# 08. UPDATE

**Goal:** Modify existing rows — single records, multiple records, atomic increments, and upserts.
**Prerequisites:** [07 — INSERT](./07-insert.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Update one row by PK | `UPDATE users SET name = 'Bob' WHERE id = 1;` | `prisma.user.update({ where: { id: 1 }, data: { name: 'Bob' } })` |
| Update one row, return it | `UPDATE users SET ... WHERE id = 1 RETURNING *;` | `update()` always returns the updated record |
| Update many rows | `UPDATE users SET role = 'moderator' WHERE role = 'user';` | `prisma.user.updateMany({ where: { role: 'user' }, data: { role: 'moderator' } })` |
| `updateMany` return value | `UPDATE ... RETURNING *` returns rows | `updateMany` returns `{ count: n }` — not individual records |
| Increment a number | `UPDATE posts SET views = views + 1 WHERE id = 1;` | `prisma.post.update({ where: { id: 1 }, data: { views: { increment: 1 } } })` |
| Decrement | `UPDATE posts SET likes = likes - 1 WHERE id = 1;` | `data: { likes: { decrement: 1 } }` |
| Multiply | `UPDATE products SET price = price * 1.1 WHERE ...;` | `data: { price: { multiply: 1.1 } }` |
| Divide | `UPDATE products SET price = price / 2 WHERE ...;` | `data: { price: { divide: 2 } }` |
| Upsert | `INSERT ... ON CONFLICT (...) DO UPDATE SET ...` | `prisma.user.upsert({ where, create, update })` |
| Update a nullable field to NULL | `UPDATE users SET bio = NULL WHERE id = 1;` | `data: { bio: null }` |
| Update nested relation | Multi-step SQL with FK | `prisma.user.update({ where: {...}, data: { posts: { create: {...} } } })` |
| Conditional update | `UPDATE ... WHERE condition` (standard SQL) | `updateMany({ where: {...}, data: {...} })` |

### Code: Update one row

```sql
-- SQL
UPDATE users
SET name = 'Robert', role = 'moderator'
WHERE id = 1
RETURNING *;
```

```js
// Prisma
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    name: 'Robert',
    role: 'moderator'
  }
})
// Returns the full updated User object
```

### Code: Update many rows

```sql
-- SQL: publish all draft posts by author 5
UPDATE posts
SET published = true
WHERE author_id = 5 AND published = false;
```

```js
// Prisma
const result = await prisma.post.updateMany({
  where: { authorId: 5, published: false },
  data: { published: true }
})
// Returns: { count: n }
```

### Code: Atomic increment

```sql
-- SQL: increment view count (safe from race conditions)
UPDATE posts
SET views = views + 1
WHERE id = 42
RETURNING views;
```

```js
// Prisma
const post = await prisma.post.update({
  where: { id: 42 },
  data: { views: { increment: 1 } }
})
// Returns the post with the new views count
```

### Code: Upsert (create or update)

```sql
-- SQL
INSERT INTO users (name, email, role)
VALUES ('Alice', 'alice@example.com', 'admin')
ON CONFLICT (email)
DO UPDATE SET role = EXCLUDED.role;
```

```js
// Prisma
const user = await prisma.user.upsert({
  where:  { email: 'alice@example.com' },
  create: { name: 'Alice', email: 'alice@example.com', role: 'admin' },
  update: { role: 'admin' }
})
```

---

## What's Really Happening

**`update()` vs `updateMany()`** — `update()` expects a `where` that uniquely identifies one row (a `@id` or `@unique` field). If no row matches, Prisma throws `P2025 Record not found`. `updateMany()` accepts any `where` condition, affects zero or more rows, and returns a count.

**Atomic operations** (`increment`, `decrement`, `multiply`, `divide`) — these compile to SQL like `views = views + $1`. They run inside the database, so two concurrent requests both incrementing the same counter won't lose updates (unlike a read-then-write pattern: `fetch views → add 1 → save`). Always use these for counters.

**`update()` always returns the record** — similar to `create()`, Prisma's `update()` internally adds `RETURNING *`. This is a round-trip cost: if you're doing bulk updates and don't need the data back, use `updateMany()` instead.

**`updateMany` and `where`** — `updateMany` with no `where` clause updates every row in the table. This is equivalent to `UPDATE posts SET published = true` — it's intentional but easy to do accidentally. Always double-check your `where` before running `updateMany` in production.

**Nested updates** — you can connect, disconnect, create, or delete related records inside an `update()` call. For example: `prisma.user.update({ where: { id: 1 }, data: { posts: { create: { title: 'New post' } } } })`. These are wrapped in an implicit transaction.

> **Leaky abstraction alert:** Prisma's `update()` throws `P2025` if the record doesn't exist. SQL's `UPDATE ... WHERE id = 1` silently updates zero rows and succeeds. If you're migrating logic from raw SQL, check for this behavior difference — code that used to "succeed silently" on a missing record will now throw.

---

## Practice

1. Write a SQL `UPDATE` and Prisma `update()` that changes a user's `role` to `'admin'` by their `id`. What happens in each if the user doesn't exist?

2. Use Prisma `updateMany` to mark all posts older than 30 days as `published = false`. Write the equivalent SQL `UPDATE ... WHERE`.

3. You have a `likes` counter on posts. Write both the SQL and Prisma approaches to increment it safely. Why is `{ increment: 1 }` safer than fetching the current value and adding 1 in JavaScript?

---

### Documentation Links

- PostgreSQL `UPDATE`: [UPDATE](https://www.postgresql.org/docs/current/sql-update.html)
- Prisma `update`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#update)
- Prisma `updateMany`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#updatemany)
- Prisma atomic number operations: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#atomic-number-operations)
- Prisma `upsert`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#upsert)
