# 06. Filtering with WHERE

**Goal:** Filter query results using conditions, comparison operators, and logical combinators.
**Prerequisites:** [05 — SELECT Basics](./05-select-basics.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Equality | `WHERE role = 'admin'` | `where: { role: 'admin' }` |
| Not equal | `WHERE role != 'admin'` | `where: { role: { not: 'admin' } }` |
| Greater than | `WHERE id > 10` | `where: { id: { gt: 10 } }` |
| Greater than or equal | `WHERE id >= 10` | `where: { id: { gte: 10 } }` |
| Less than | `WHERE id < 10` | `where: { id: { lt: 10 } }` |
| Less than or equal | `WHERE id <= 10` | `where: { id: { lte: 10 } }` |
| Between (inclusive) | `WHERE id BETWEEN 10 AND 20` | `where: { id: { gte: 10, lte: 20 } }` |
| IN a list | `WHERE role IN ('admin', 'moderator')` | `where: { role: { in: ['admin', 'moderator'] } }` |
| NOT IN a list | `WHERE role NOT IN ('admin', 'moderator')` | `where: { role: { notIn: ['admin', 'moderator'] } }` |
| LIKE (case-sensitive) | `WHERE name LIKE 'Ali%'` | `where: { name: { startsWith: 'Ali' } }` |
| ILIKE (case-insensitive) | `WHERE name ILIKE '%alice%'` | `where: { name: { contains: 'alice', mode: 'insensitive' } }` |
| Contains (case-sensitive) | `WHERE name LIKE '%alice%'` | `where: { name: { contains: 'alice' } }` |
| Starts with | `WHERE name LIKE 'Al%'` | `where: { name: { startsWith: 'Al' } }` |
| Ends with | `WHERE email LIKE '%@gmail.com'` | `where: { email: { endsWith: '@gmail.com' } }` |
| IS NULL | `WHERE bio IS NULL` | `where: { bio: null }` |
| IS NOT NULL | `WHERE bio IS NOT NULL` | `where: { bio: { not: null } }` |
| AND (implicit) | `WHERE role = 'admin' AND published = true` | `where: { role: 'admin', published: true }` |
| AND (explicit) | `WHERE a = 1 AND b = 2` | `where: { AND: [{ a: 1 }, { b: 2 }] }` |
| OR | `WHERE role = 'admin' OR role = 'moderator'` | `where: { OR: [{ role: 'admin' }, { role: 'moderator' }] }` |
| NOT | `WHERE NOT published = true` | `where: { NOT: { published: true } }` |

### Code: Multiple conditions (AND)

```sql
-- SQL: find published posts by a specific author
SELECT * FROM posts
WHERE published = true AND author_id = 5;
```

```js
// Prisma
const posts = await prisma.post.findMany({
  where: {
    published: true,
    authorId: 5      // multiple keys = implicit AND
  }
})
```

### Code: OR condition

```sql
-- SQL: find admins or moderators
SELECT * FROM users
WHERE role = 'admin' OR role = 'moderator';
```

```js
// Prisma
const privileged = await prisma.user.findMany({
  where: {
    OR: [
      { role: 'admin' },
      { role: 'moderator' }
    ]
  }
})
```

### Code: Case-insensitive search

```sql
-- SQL: find users with 'alice' anywhere in name (case-insensitive)
SELECT * FROM users WHERE name ILIKE '%alice%';
```

```js
// Prisma
const users = await prisma.user.findMany({
  where: {
    name: {
      contains: 'alice',
      mode: 'insensitive'   // maps to ILIKE in PostgreSQL
    }
  }
})
```

### Code: NULL check

```sql
-- SQL: find users with no bio
SELECT * FROM users WHERE bio IS NULL;
```

```js
// Prisma
const users = await prisma.user.findMany({
  where: { bio: null }
})
```

### Code: Nested AND + OR

```sql
-- SQL: published posts, authored by admin or moderator
SELECT p.*
FROM posts p
JOIN users u ON p.author_id = u.id
WHERE p.published = true
  AND (u.role = 'admin' OR u.role = 'moderator');
```

```js
// Prisma (relation filter — covered in file 13)
const posts = await prisma.post.findMany({
  where: {
    published: true,
    author: {
      role: { in: ['admin', 'moderator'] }
    }
  }
})
```

---

## What's Really Happening

Prisma's `where` object compiles directly to SQL `WHERE` clauses. Multiple keys at the top level become `AND` conditions. The `OR` and `NOT` operators must be spelled out explicitly as arrays.

**`mode: 'insensitive'`** — this only works on PostgreSQL (it compiles to `ILIKE`). On other databases it may be silently ignored or error. Since you're on Supabase/Postgres, it's safe to use.

**`BETWEEN` has no Prisma equivalent** — use `{ gte: x, lte: y }` instead. They generate the same query plan.

**`LIKE` with `%` wildcards** — Prisma's `contains`, `startsWith`, and `endsWith` map to `LIKE '%x%'`, `LIKE 'x%'`, and `LIKE '%x'` respectively. There's no way to write an arbitrary LIKE pattern (e.g. `'A_ice%'` with a single-character wildcard `_`). For that, use `$queryRaw`.

**Filtering on relation fields** — the last example above filters on `author.role`, which traverses the `posts → users` relationship. Prisma compiles this to a JOIN or subquery. This is covered deeply in [13 — JOINs vs Includes](./13-joins-vs-includes.md).

> **Leaky abstraction alert:** `mode: 'insensitive'` generates `ILIKE` in Postgres, which cannot use standard B-tree indexes. For case-insensitive search at scale, create a functional index: `CREATE INDEX ON users (LOWER(name));` and query `WHERE LOWER(name) LIKE LOWER('%alice%')`.

---

## Practice

1. Write a SQL query and Prisma equivalent that finds all posts where `published = false` and `created_at` is before a given date.

2. Write a SQL query and Prisma equivalent that finds all users whose email ends with `@example.com` OR whose role is `admin`.

3. Write a SQL query using `ILIKE` to find users with 'smith' in their name. Then write the Prisma equivalent with `mode: 'insensitive'`. What index would you add to make this fast at scale?

---

### Documentation Links

- PostgreSQL `WHERE` conditions: [Row Expressions](https://www.postgresql.org/docs/current/functions-comparison.html)
- PostgreSQL `LIKE` / `ILIKE`: [Pattern Matching](https://www.postgresql.org/docs/current/functions-matching.html)
- Prisma filtering: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/filtering-and-sorting)
- Prisma filter reference: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#filter-conditions-and-operators)
