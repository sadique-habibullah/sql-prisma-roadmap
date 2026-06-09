# 05. SELECT Basics

**Goal:** Read rows from a table — all columns, specific columns, by primary key, and by first match.
**Prerequisites:** [04 — Tables & Models](./04-tables-and-models.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Select all rows, all columns | `SELECT * FROM users;` | `prisma.user.findMany()` |
| Select specific columns | `SELECT id, name, email FROM users;` | `prisma.user.findMany({ select: { id: true, name: true, email: true } })` |
| Select one row by primary key | `SELECT * FROM users WHERE id = 1;` | `prisma.user.findUnique({ where: { id: 1 } })` |
| Select first matching row | `SELECT * FROM users WHERE role = 'admin' LIMIT 1;` | `prisma.user.findFirst({ where: { role: 'admin' } })` |
| Count all rows | `SELECT COUNT(*) FROM users;` | `prisma.user.count()` |
| Count with condition | `SELECT COUNT(*) FROM users WHERE role = 'admin';` | `prisma.user.count({ where: { role: 'admin' } })` |
| Computed/aliased column | `SELECT name, LENGTH(name) AS name_length FROM users;` | No equivalent — use `$queryRaw` |
| Distinct values | `SELECT DISTINCT role FROM users;` | `prisma.user.findMany({ select: { role: true }, distinct: ['role'] })` |
| Alias table | `SELECT u.name FROM users u;` | Not applicable — Prisma handles aliases internally |
| All rows with limit | `SELECT * FROM users LIMIT 5;` | `prisma.user.findMany({ take: 5 })` |

### Code: Select all users

```sql
-- SQL
SELECT * FROM users;
```

```js
// Prisma
const users = await prisma.user.findMany()
// Returns: User[]
```

### Code: Select specific fields

```sql
-- SQL
SELECT id, name, email FROM users;
```

```js
// Prisma
const users = await prisma.user.findMany({
  select: { id: true, name: true, email: true }
})
// Returns: { id: number, name: string, email: string }[]
// Note: fields NOT in select are absent from the result object
```

### Code: Find by primary key

```sql
-- SQL
SELECT * FROM users WHERE id = 1;
```

```js
// Prisma
const user = await prisma.user.findUnique({
  where: { id: 1 }
})
// Returns: User | null  (null if no row with id=1 exists)
```

### Code: Find first match

```sql
-- SQL
SELECT * FROM users WHERE role = 'admin' LIMIT 1;
```

```js
// Prisma
const admin = await prisma.user.findFirst({
  where: { role: 'admin' }
})
// Returns: User | null
```

### Code: Count

```sql
-- SQL
SELECT COUNT(*) FROM users;
```

```js
// Prisma
const total = await prisma.user.count()
// Returns: number  (not BigInt — Prisma converts COUNT to a JS number)
```

---

## What's Really Happening

**`findUnique` vs `findFirst`** — these look similar but have a meaningful difference:

- `findUnique` only accepts fields marked `@id` or `@unique` in the `where` clause. It compiles to `WHERE id = $1 LIMIT 1` and Postgres uses the primary key index — guaranteed fast.
- `findFirst` accepts any field in `where`. It compiles to `WHERE role = $1 LIMIT 1` and may do a full table scan if there's no index on `role`.

Use `findUnique` when you have a unique identifier. Use `findFirst` only when you genuinely want "any one row matching this condition."

**`select` in Prisma** — when you add a `select` clause, Prisma generates `SELECT id, name, email FROM users` instead of `SELECT *`. This is a real performance optimization: you're not fetching columns you don't need. However, the return type changes — a `select`ed result is NOT a full `User` object; it's a typed subset. TypeScript will catch you if you try to access a field you didn't select.

**`SELECT *` is usually a code smell** — in production SQL, `SELECT *` is fragile: adding a large `JSONB` column to a table suddenly slows down every query that used `*`. Always select the columns you need. The Prisma equivalent is explicit `select: {}`.

**Count returns a number** — `SELECT COUNT(*) FROM users` returns a `bigint` in PostgreSQL (because counts could theoretically exceed `INT` range). Prisma's `count()` safely converts this to a JavaScript `number`, which is fine for any practical row count.

---

## Practice

1. Write a SQL query and Prisma call that returns only the `email` and `role` of all users with `role = 'moderator'`.

2. What's the difference between `findUnique({ where: { id: 1 } })` and `findFirst({ where: { id: 1 } })`? Write both, then check the SQL Prisma generates with `prisma.$queryRaw`.

3. Write a SQL query with a computed column: `SELECT id, name, UPPER(name) AS name_upper FROM users`. Why can't Prisma express this directly? What's your workaround?

---

### Documentation Links

- PostgreSQL `SELECT`: [SELECT](https://www.postgresql.org/docs/current/sql-select.html)
- PostgreSQL `COUNT`: [Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)
- Prisma `findMany`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#findmany)
- Prisma `findUnique`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#findunique)
- Prisma `findFirst`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#findfirst)
- Prisma `select`: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/select-fields)
