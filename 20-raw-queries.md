# 20. Raw Queries

**Goal:** Drop to raw SQL using Prisma's escape hatches when the query API can't express what you need — safely, without SQL injection.
**Prerequisites:** [05 — SELECT Basics](./05-select-basics.md), [16 — Transactions](./16-transactions.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Read rows with raw SQL | `SELECT * FROM users WHERE id = $1` | `` prisma.$queryRaw`SELECT * FROM users WHERE id = ${id}` `` |
| Execute DML / DDL | `UPDATE users SET name = $1 WHERE id = $2` | `` prisma.$executeRaw`UPDATE users SET name = ${name} WHERE id = ${id}` `` |
| Safe parameterization | `$1`, `$2` placeholders | Tagged template literal auto-parameterizes interpolations |
| Manual parameterization | `SELECT * FROM users WHERE id = $1` with `[id]` array | `prisma.$queryRawUnsafe('SELECT * FROM users WHERE id = $1', id)` |
| Dangerous raw string | Concatenating SQL strings (SQL injection risk) | `prisma.$queryRawUnsafe(sql)` with no params — **avoid** |
| Type the result | `row.name as string` in your head | `` prisma.$queryRaw<User[]>`SELECT ...` `` — TypeScript generic |
| Build dynamic SQL safely | `format('SELECT ... WHERE %I = %L', col, val)` | `Prisma.sql` tagged template for composable fragments |
| Raw query in transaction | `BEGIN; SELECT ...; COMMIT;` | `tx.$queryRaw` / `tx.$executeRaw` inside `$transaction` callback |
| Return count from DML | `UPDATE ... RETURNING id` | `$executeRaw` returns affected row count as a number |

### Code: Safe read — tagged template (recommended)

```sql
-- SQL
SELECT * FROM users WHERE email = 'alice@example.com';
```

```js
// Prisma: tagged template automatically parameterizes
const email = 'alice@example.com'

const users = await prisma.$queryRaw`
  SELECT id, name, email FROM users
  WHERE email = ${email}
`
// Compiled to: SELECT ... WHERE email = $1  with ['alice@example.com']
// SQL injection is NOT possible through template interpolations
```

### Code: Typed result

```js
// Specify the expected return type with a TypeScript generic
type UserRow = { id: number; name: string; email: string }

const users = await prisma.$queryRaw<UserRow[]>`
  SELECT id, name, email FROM users
  WHERE role = ${'admin'}
`
// users is typed as UserRow[]
```

### Code: $executeRaw (for INSERT / UPDATE / DELETE / DDL)

```js
// Returns number of affected rows
const count = await prisma.$executeRaw`
  UPDATE posts
  SET published = true
  WHERE created_at < NOW() - INTERVAL '1 year'
`
console.log(`${count} posts published`)
```

### Code: Composable SQL with `Prisma.sql`

```js
import { Prisma } from '@prisma/client'

// Build fragments safely — each fragment is parameterized
const roleFilter = Prisma.sql`AND role = ${'admin'}`
const limit     = Prisma.sql`LIMIT ${10}`

const users = await prisma.$queryRaw`
  SELECT id, name, email
  FROM users
  WHERE 1 = 1
  ${roleFilter}
  ORDER BY created_at DESC
  ${limit}
`
```

### Code: $queryRawUnsafe (for dynamic column/table names)

```js
// Use $queryRawUnsafe ONLY when you must interpolate identifiers
// (column names, table names) that can't be parameterized.
// NEVER put user input directly here — validate it first.

const allowedColumns = ['name', 'email', 'created_at']
const sortColumn = 'name'  // from user input — MUST validate before use

if (!allowedColumns.includes(sortColumn)) {
  throw new Error('Invalid sort column')
}

// The column name can't be a $1 parameter — so use $queryRawUnsafe
const users = await prisma.$queryRawUnsafe(
  `SELECT id, name, email FROM users ORDER BY ${sortColumn} ASC`
)
// This is safe ONLY because we validated sortColumn against an allowlist
```

### Code: Raw query inside a transaction

```js
await prisma.$transaction(async (tx) => {
  // Use tx.$queryRaw / tx.$executeRaw inside transactions
  const posts = await tx.$queryRaw<{ id: number }[]>`
    SELECT id FROM posts WHERE published = false FOR UPDATE
  `

  for (const post of posts) {
    await tx.$executeRaw`
      UPDATE posts SET published = true WHERE id = ${post.id}
    `
  }
})
```

---

## What's Really Happening

**Tagged templates prevent SQL injection** — when you write `` prisma.$queryRaw`SELECT * FROM users WHERE id = ${userId}` ``, Prisma does NOT concatenate `userId` into the string. It creates a parameterized query: `SELECT * FROM users WHERE id = $1` with `[userId]` as the parameters array. The database never sees the raw value as part of the query string.

**You cannot parameterize identifiers** — SQL parameters (`$1`, `$2`) can only substitute *values* (strings, numbers, dates). They cannot substitute column names, table names, or SQL keywords. `SELECT * FROM $1 WHERE $2 = $3` is invalid SQL. If you need a dynamic column name, use `$queryRawUnsafe` with strict allowlist validation.

**`$executeRaw` returns affected row count** — it returns a `number` (or `bigint` in some versions) representing how many rows were affected by an `UPDATE`, `DELETE`, or `INSERT`. It does NOT return the rows themselves. For DDL (`CREATE TABLE`, `ALTER TABLE`, etc.) it returns 0.

**`Prisma.sql` for fragments** — `Prisma.sql` is a tagged template that creates a `Sql` object (not a string). You can embed `Prisma.sql` fragments inside other `Prisma.sql` or `$queryRaw` templates. Each fragment's interpolations are still parameterized. This is the safe way to build dynamic WHERE clauses or ORDER BY clauses.

**BigInt from COUNT** — raw queries that use `COUNT(*)` return a PostgreSQL `bigint`, which becomes a JavaScript `BigInt`. Cast it in SQL (`COUNT(*)::int`) if you want a regular JS number.

> **Leaky abstraction alert:** `$queryRaw` returns an array of plain objects, not Prisma model instances. There's no relation loading, no `@map` transformation, and no `@db` type conversion. Column names match exactly what the database returns (often snake_case). You must handle type casting yourself — e.g., `timestamp` columns come back as JavaScript `Date` objects, but `numeric`/`decimal` columns come back as strings in some versions of the pg driver.

---

## Practice

1. Write a `$queryRaw` that fetches all posts with a body longer than 500 characters using the SQL `LENGTH()` function. Type the result with a TypeScript interface.

2. Write a `$executeRaw` that deletes all posts older than 1 year that are not published. Verify the returned count.

3. A user can choose a sort field from a dropdown (`name`, `email`, `createdAt`). Write a safe function using `$queryRawUnsafe` with allowlist validation that handles this sorting. Why can't you use the tagged template approach for the column name?

---

### Documentation Links

- PostgreSQL parameterized queries: [libpq extended query protocol](https://www.postgresql.org/docs/current/libpq-exec.html)
- PostgreSQL SQL injection prevention: [Security](https://www.postgresql.org/docs/current/prevent-password-guessing.html)
- Prisma `$queryRaw`: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access/raw-queries#queryraw)
- Prisma `$executeRaw`: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access/raw-queries#executeraw)
- Prisma `Prisma.sql`: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access/raw-queries#tagged-template-helpers)
- OWASP SQL injection prevention: (verify: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
