# 17. JSON, Arrays & Enums

**Goal:** Work with semi-structured data (JSONB), array columns, and fixed value sets (enums) in both SQL and Prisma.
**Prerequisites:** [03 — Data Types](./03-data-types.md), [04 — Tables & Models](./04-tables-and-models.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| JSONB column | `metadata JSONB` | `metadata Json` |
| JSON column | `data JSON` | `metadata Json @db.Json` |
| Read a JSONB field | `metadata->>'role'` | `where: { metadata: { path: ['role'], equals: 'admin' } }` |
| Read nested JSONB | `metadata#>>'{address,city}'` | `where: { metadata: { path: ['address', 'city'], equals: 'NYC' } }` |
| JSONB contains | `metadata @> '{"role":"admin"}'` | **No direct equivalent** — use `$queryRaw` |
| JSONB is contained by | `metadata <@ '{"role":"admin","active":true}'` | **No direct equivalent** — use `$queryRaw` |
| JSONB has key | `metadata ? 'role'` | **No direct equivalent** — use `$queryRaw` |
| Native array column | `tags TEXT[]` | `tags String[]` |
| Array contains element | `WHERE 'tag1' = ANY(tags)` | `where: { tags: { has: 'tag1' } }` |
| Array has every element | `WHERE tags @> ARRAY['a','b']` | `where: { tags: { hasEvery: ['a', 'b'] } }` |
| Array has some element | `WHERE tags && ARRAY['a','b']` | `where: { tags: { hasSome: ['a', 'b'] } }` |
| Array is empty | `WHERE tags = '{}'` | `where: { tags: { isEmpty: true } }` |
| Postgres ENUM type | `CREATE TYPE role AS ENUM ('admin', 'user')` | `enum Role { admin user }` in schema |
| Enum column | `role user_role NOT NULL DEFAULT 'user'` | `role Role @default(user)` |
| Add enum value | `ALTER TYPE role ADD VALUE 'moderator';` | Add to `enum` block + `prisma migrate dev` |

### Code: JSONB column definition and schema

```sql
-- SQL
CREATE TABLE posts (
  id       SERIAL PRIMARY KEY,
  title    TEXT   NOT NULL,
  metadata JSONB
);

-- Insert with JSONB
INSERT INTO posts (title, metadata)
VALUES ('Hello', '{"views": 0, "tags": ["intro", "tutorial"], "author": {"id": 1}}');
```

```prisma
// Prisma schema
model Post {
  id       Int    @id @default(autoincrement())
  title    String
  metadata Json?
}
```

```js
// Prisma create with JSONB
await prisma.post.create({
  data: {
    title: 'Hello',
    metadata: {
      views: 0,
      tags: ['intro', 'tutorial'],
      author: { id: 1 }
    }
  }
})
```

### Code: Filtering on JSONB fields

```sql
-- SQL: find posts where metadata->>'status' = 'featured'
SELECT * FROM posts WHERE metadata->>'status' = 'featured';

-- SQL: find posts where metadata.views > 100
SELECT * FROM posts WHERE (metadata->>'views')::int > 100;
```

```js
// Prisma: path-based equality
const posts = await prisma.post.findMany({
  where: {
    metadata: {
      path: ['status'],
      equals: 'featured'
    }
  }
})
// For numeric comparisons or complex JSONB operators, use $queryRaw
```

### Code: Array columns

```sql
-- SQL: posts table with a native text array
ALTER TABLE posts ADD COLUMN tags TEXT[] NOT NULL DEFAULT '{}';

-- Insert with array
UPDATE posts SET tags = ARRAY['intro', 'tutorial'] WHERE id = 1;

-- Query: posts with the 'intro' tag
SELECT * FROM posts WHERE 'intro' = ANY(tags);
```

```prisma
// Prisma schema
model Post {
  id   Int      @id @default(autoincrement())
  tags String[]
}
```

```js
// Prisma: create with array
await prisma.post.create({
  data: { title: 'Hello', tags: ['intro', 'tutorial'] }
})

// Prisma: filter — post has the 'intro' tag
const posts = await prisma.post.findMany({
  where: { tags: { has: 'intro' } }
})
```

### Code: Enum type

```sql
-- SQL: define and use an enum type
CREATE TYPE user_role AS ENUM ('admin', 'user', 'moderator');

CREATE TABLE users (
  id   SERIAL    PRIMARY KEY,
  role user_role NOT NULL DEFAULT 'user'
);

-- Add a new value later
ALTER TYPE user_role ADD VALUE 'superadmin';
```

```prisma
// Prisma: enum definition
enum Role {
  admin
  user
  moderator
}

model User {
  id   Int    @id @default(autoincrement())
  role Role   @default(user)
}
```

```js
// Prisma: using enum values
await prisma.user.create({
  data: { name: 'Alice', email: 'a@x.com', role: 'admin' }
})

const admins = await prisma.user.findMany({
  where: { role: 'admin' }
})
```

---

## What's Really Happening

**JSONB vs JSON in Postgres** — always prefer `JSONB`. JSON stores text verbatim (preserving whitespace, key order). JSONB stores a parsed binary representation that is faster to query, can be indexed with GIN, and deduplicates keys. Prisma's `Json` type maps to `JSONB` by default.

**Prisma's JSONB filter support is limited** — Prisma supports path-based equality (`path`, `equals`) and basic checks, but the full power of PostgreSQL's JSONB operators (`@>`, `<@`, `?`, `?|`, `?&`) requires `$queryRaw`. For complex JSONB querying, write the SQL.

**Scalar lists (arrays) require PostgreSQL** — `String[]`, `Int[]`, etc. are only supported by Prisma when your `datasource.provider = "postgresql"`. They're not available on MySQL or SQLite.

**Enums in Prisma vs Postgres** — when you define an `enum` in Prisma, it creates a native Postgres `ENUM` type via migration. This means adding new enum values requires a migration (`ALTER TYPE ... ADD VALUE`). You can't arbitrarily add values at runtime. Some teams prefer a `String` column with application-level validation for more flexibility.

**Adding enum values** — `ALTER TYPE ... ADD VALUE 'newvalue'` in Postgres is a one-way operation: you can add values but cannot remove them without complex workarounds. Think carefully about your enum values before deploying.

> **Leaky abstraction alert:** Prisma's `Json` type returns raw JavaScript objects/arrays — Prisma does no type narrowing. TypeScript types the field as `Prisma.JsonValue` (a union of all possible JSON shapes). You must cast or validate the shape yourself. Consider using a validation library (Zod, TypeBox) to parse the JSON into a typed schema before using it in your application.

---

## Practice

1. Add a `metadata Json?` field to the `Post` model. Insert a post with metadata `{ views: 0, featured: false }`. Then write a Prisma query that finds posts where `metadata.featured = true`.

2. Add a `tags String[]` field to the `Post` model. Write a Prisma query that finds all posts tagged with both `"tutorial"` and `"beginner"`. Write the SQL equivalent.

3. Add a new enum value `'superadmin'` to the `Role` enum. What SQL does `prisma migrate dev` generate? What happens to existing rows in the `users` table?

---

### Documentation Links

- PostgreSQL JSONB: [JSON Types](https://www.postgresql.org/docs/current/datatype-json.html)
- PostgreSQL JSON functions and operators: [JSON Functions](https://www.postgresql.org/docs/current/functions-json.html)
- PostgreSQL arrays: [Array Types](https://www.postgresql.org/docs/current/arrays.html)
- PostgreSQL enum types: [Enumerated Types](https://www.postgresql.org/docs/current/datatype-enum.html)
- Prisma JSON fields: (verify: https://www.prisma.io/docs/orm/prisma-client/special-fields-and-types/working-with-json-fields)
- Prisma scalar lists (arrays): (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/models#scalar-lists)
- Prisma enums: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/models#defining-enums)
