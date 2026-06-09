# 07. INSERT

**Goal:** Add single rows, multiple rows, handle conflicts, and write nested creates that span related tables.
**Prerequisites:** [06 — Filtering with WHERE](./06-filtering-where.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Insert one row | `INSERT INTO users (name, email) VALUES ('Alice', 'a@x.com');` | `prisma.user.create({ data: { name: 'Alice', email: 'a@x.com' } })` |
| Insert and return the row | `INSERT INTO users (...) VALUES (...) RETURNING *;` | `create()` always returns the full created record |
| Insert many rows | `INSERT INTO users (name, email) VALUES (...), (...), (...);` | `prisma.user.createMany({ data: [...] })` |
| `createMany` return value | `INSERT ... RETURNING *` returns each row | `createMany` returns `{ count: n }` — not individual records |
| Skip duplicate rows | `INSERT ... ON CONFLICT DO NOTHING` | `prisma.user.createMany({ data: [...], skipDuplicates: true })` |
| Upsert (insert or update) | `INSERT ... ON CONFLICT (...) DO UPDATE SET ...` | `prisma.user.upsert({ where, create, update })` |
| Insert with nested relation | Two separate `INSERT` statements with FK | `prisma.user.create({ data: { posts: { create: { title: '...' } } } })` |
| Insert returning specific columns | `INSERT ... RETURNING id, name` | `prisma.user.create({ data: {...}, select: { id: true, name: true } })` |
| Insert with default values | `INSERT INTO users (name) VALUES ('Bob')` (email has DEFAULT) | `prisma.user.create({ data: { name: 'Bob' } })` — omit fields with defaults |

### Code: Insert one user

```sql
-- SQL
INSERT INTO users (name, email, role)
VALUES ('Alice', 'alice@example.com', 'user')
RETURNING *;
```

```js
// Prisma
const user = await prisma.user.create({
  data: {
    name: 'Alice',
    email: 'alice@example.com',
    role: 'user'
  }
})
// Returns the full User object (always, no RETURNING needed)
```

### Code: Insert many users

```sql
-- SQL
INSERT INTO users (name, email)
VALUES
  ('Bob',     'bob@example.com'),
  ('Charlie', 'charlie@example.com'),
  ('Diana',   'diana@example.com');
```

```js
// Prisma
const result = await prisma.user.createMany({
  data: [
    { name: 'Bob',     email: 'bob@example.com' },
    { name: 'Charlie', email: 'charlie@example.com' },
    { name: 'Diana',   email: 'diana@example.com' }
  ]
})
// Returns: { count: 3 }  — NOT the created records
```

### Code: Upsert

```sql
-- SQL: insert or update if email already exists
INSERT INTO users (name, email)
VALUES ('Alice Updated', 'alice@example.com')
ON CONFLICT (email)
DO UPDATE SET name = EXCLUDED.name;
```

```js
// Prisma
const user = await prisma.user.upsert({
  where:  { email: 'alice@example.com' },
  create: { name: 'Alice Updated', email: 'alice@example.com' },
  update: { name: 'Alice Updated' }
})
// Returns the created or updated record
```

### Code: Nested create (user + post in one call)

```sql
-- SQL: two separate inserts
INSERT INTO users (name, email) VALUES ('Eve', 'eve@example.com') RETURNING id;
-- (use returned id in the next statement)
INSERT INTO posts (title, body, author_id) VALUES ('Hello World', 'My first post', <returned_id>);
```

```js
// Prisma: one nested call, wrapped in an implicit transaction
const user = await prisma.user.create({
  data: {
    name: 'Eve',
    email: 'eve@example.com',
    posts: {
      create: {
        title: 'Hello World',
        body: 'My first post'
      }
    }
  },
  include: { posts: true }
})
```

---

## What's Really Happening

**`create()` always returns the record** — in SQL, `INSERT` does not return data unless you add `RETURNING`. In Prisma, `create()` always performs a `RETURNING *` internally, so you always get the full object back. Use `select` to limit which fields come back if the model is large.

**`createMany()` does not return records** — this is a deliberate performance tradeoff. Prisma's `createMany` compiles to a single `INSERT ... VALUES (...), (...), (...)` statement and returns only a count. If you need the created records, either use individual `create()` calls in a `$transaction`, or follow up with a `findMany`.

**`skipDuplicates`** compiles to `ON CONFLICT DO NOTHING`, not `ON CONFLICT DO UPDATE`. Any row that would violate a unique constraint is silently skipped, and the count reflects how many rows were actually inserted.

**Nested `create`** — when you nest `posts: { create: { ... } }` inside a `user.create()`, Prisma wraps both inserts in a database transaction automatically. If the post insert fails, the user insert is also rolled back. This saves you from writing the `BEGIN/COMMIT` yourself.

**`upsert` requires a unique field in `where`** — Prisma compiles `upsert` to `INSERT ... ON CONFLICT (...) DO UPDATE SET ...`. The `where` clause must reference a `@unique` or `@id` field. If it doesn't, Prisma throws at runtime.

> **Leaky abstraction alert:** `createMany` is not supported in SQLite (different Prisma provider). On PostgreSQL it works correctly. Also, nested `createMany` (creating many related records at once) has some limitations — check Prisma docs for the specific supported nestings.

---

## Practice

1. Write a SQL `INSERT` and Prisma `create()` that adds a new post with `title`, `body`, and `authorId`. Make the Prisma call return only `id` and `title`.

2. Insert 3 tags using `createMany`. What does the return value look like? How would you retrieve the created tags afterward?

3. Write a SQL `ON CONFLICT DO UPDATE` and the equivalent Prisma `upsert` that creates a user if the email doesn't exist, or updates the `name` if it does.

---

### Documentation Links

- PostgreSQL `INSERT`: [INSERT](https://www.postgresql.org/docs/current/sql-insert.html)
- PostgreSQL `ON CONFLICT`: [INSERT — ON CONFLICT clause](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT)
- Prisma `create`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#create)
- Prisma `createMany`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#createmany)
- Prisma `upsert`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#upsert)
- Prisma nested writes: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries#nested-writes)
