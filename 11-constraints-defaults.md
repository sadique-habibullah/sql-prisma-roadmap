# 11. Constraints & Defaults

**Goal:** Enforce data integrity at the database level using constraints and default values.
**Prerequisites:** [04 — Tables & Models](./04-tables-and-models.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Primary key | `id SERIAL PRIMARY KEY` | `id Int @id @default(autoincrement())` |
| UUID primary key | `id UUID DEFAULT gen_random_uuid() PRIMARY KEY` | `id String @id @default(uuid()) @db.Uuid` |
| Unique (single column) | `email TEXT UNIQUE` | `email String @unique` |
| Unique (composite) | `UNIQUE (tenant_id, slug)` | `@@unique([tenantId, slug])` |
| NOT NULL | `name TEXT NOT NULL` | `name String` (any non-optional field = NOT NULL) |
| Nullable | `bio TEXT` | `bio String?` |
| Default literal value | `role TEXT DEFAULT 'user'` | `role String @default("user")` |
| Default current timestamp | `created_at TIMESTAMPTZ DEFAULT NOW()` | `createdAt DateTime @default(now())` |
| Default UUID | `id UUID DEFAULT gen_random_uuid()` | `id String @default(uuid())` |
| Auto-increment default | `id SERIAL` | `id Int @default(autoincrement())` |
| CUID default | No SQL equivalent | `id String @id @default(cuid())` |
| CHECK constraint | `CHECK (age >= 0)` | **No Prisma equivalent** — add via `$executeRaw` in migration |
| EXCLUDE constraint | `EXCLUDE USING gist (...)` | **No Prisma equivalent** — add via `$executeRaw` in migration |
| Named constraint | `CONSTRAINT unique_email UNIQUE (email)` | Prisma generates constraint names automatically |
| Deferrable constraint | `DEFERRABLE INITIALLY DEFERRED` | **No Prisma equivalent** — add via `$executeRaw` |

### Code: Primary key types

```sql
-- SQL: auto-increment integer PK
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  ...
);

-- SQL: UUID PK
CREATE TABLE posts (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  ...
);
```

```prisma
// Prisma: auto-increment
model Post {
  id Int @id @default(autoincrement())
}

// Prisma: UUID
model Post {
  id String @id @default(uuid()) @db.Uuid
}
```

### Code: Unique constraints

```sql
-- Single column
CREATE TABLE users (
  email TEXT UNIQUE
);

-- Composite unique
CREATE TABLE memberships (
  user_id    INT REFERENCES users(id),
  org_id     INT REFERENCES organizations(id),
  UNIQUE (user_id, org_id)
);
```

```prisma
// Single column
model User {
  email String @unique
}

// Composite unique
model Membership {
  userId Int
  orgId  Int
  @@unique([userId, orgId])
}
```

### Code: CHECK constraint (Prisma workaround)

```sql
-- SQL: enforce age >= 0 at database level
ALTER TABLE users ADD CONSTRAINT check_age CHECK (age >= 0);
```

```js
// Prisma: no schema-level equivalent
// Add it in a custom migration:
await prisma.$executeRaw`
  ALTER TABLE users
  ADD CONSTRAINT check_age CHECK (age >= 0)
`
// Or add it manually in the generated migration SQL file
```

### Code: Defaults

```sql
-- SQL: various defaults
CREATE TABLE articles (
  id         SERIAL      PRIMARY KEY,
  status     TEXT        NOT NULL DEFAULT 'draft',
  view_count INTEGER     NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```prisma
// Prisma equivalents
model Article {
  id        Int      @id @default(autoincrement())
  status    String   @default("draft")
  viewCount Int      @default(0)
  createdAt DateTime @default(now())
}
```

---

## What's Really Happening

**Constraints are database-level guards** — they fire even if a row is inserted via the Supabase dashboard, a raw SQL script, or any other connection bypassing Prisma. Prisma-level validations (if you add them manually in application code) only protect you when going through Prisma. Prefer database constraints for correctness.

**`NOT NULL` via non-optional fields** — in Prisma, a field without `?` generates `NOT NULL` in SQL. Beginners often expect `?` to mean "optional to provide in create()". It doesn't — it means the database allows NULL. To make a field optional in `create()`, give it a `@default`.

**`@unique` vs `@@unique`** — `@unique` is a single-column unique constraint (on the field). `@@unique([a, b])` is a composite constraint — the *combination* of `a` and `b` must be unique, not each individually.

**CHECK constraints** — these are the one common constraint type Prisma cannot generate. You have two options:
1. Write the `ALTER TABLE ... ADD CONSTRAINT ...` SQL directly in the Prisma-generated migration file (Prisma won't overwrite it on the next migration, but `prisma migrate reset` will drop it).
2. Run it as a `$executeRaw` call in a data migration or seed script.

**`@default(cuid())`** — Prisma can generate CUIDs (compact unique IDs) client-side, similar to `uuid()`. CUIDs are sortable and URL-safe. Unlike `uuid()`, there is no native Postgres function for CUID — Prisma generates the value in JavaScript before inserting.

> **Leaky abstraction alert:** When Prisma generates UUID values with `@default(uuid())`, it generates them in the application (client-side), not in the database. This means if you insert rows via raw SQL without specifying the `id`, you get a `NOT NULL` violation — the database has no default. To have the database generate UUIDs, use `@db.Uuid` with a migration that sets `DEFAULT gen_random_uuid()`.

---

## Practice

1. Design a `memberships` table (SQL + Prisma model) where the combination of `userId` and `organizationId` must be unique. What SQL constraint does Prisma generate?

2. Add a `CHECK (price > 0)` constraint to a `products` table. Write the `$executeRaw` call or the raw SQL to add it. Why doesn't Prisma support this in the schema file?

3. What is the difference between `@default(uuid())` (Prisma-side) and using `@db.Uuid` with `DEFAULT gen_random_uuid()` (database-side)? When would you prefer each?

---

### Documentation Links

- PostgreSQL constraints: [Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
- PostgreSQL `DEFAULT`: [Default Values](https://www.postgresql.org/docs/current/ddl-default.html)
- Prisma `@id`, `@unique`, `@default`: (verify: https://www.prisma.io/docs/orm/reference/prisma-schema-reference#attributes)
- Prisma `@@unique` composite: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/models#defining-a-unique-field)
