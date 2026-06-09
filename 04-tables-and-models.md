# 04. Tables & Models

**Goal:** Translate `CREATE TABLE` to Prisma `model`, and establish the unified schema used throughout this entire roadmap.
**Prerequisites:** [03 — Data Types](./03-data-types.md)

---

## The Unified Schema

Every file from here onward uses this schema. Learn its shape now.

```
users      (id, name, email, role, created_at)
posts      (id, title, body, published, author_id → users.id, created_at)
comments   (id, body, author_id → users.id, post_id → posts.id, created_at)
tags       (id, name)
post_tags  (post_id → posts.id, tag_id → tags.id)   -- many-to-many join table
```

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Create a table | `CREATE TABLE users (...)` | `model User { ... }` in `schema.prisma` |
| Integer primary key | `id SERIAL PRIMARY KEY` | `id Int @id @default(autoincrement())` |
| UUID primary key | `id UUID DEFAULT gen_random_uuid() PRIMARY KEY` | `id String @id @default(uuid()) @db.Uuid` |
| Required text column | `name TEXT NOT NULL` | `name String` (non-optional = NOT NULL) |
| Optional text column | `bio TEXT` | `bio String?` (the `?` means nullable) |
| Boolean with default | `published BOOLEAN NOT NULL DEFAULT false` | `published Boolean @default(false)` |
| Timestamp with default | `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()` | `createdAt DateTime @default(now())` |
| Auto-update timestamp | Requires a trigger in SQL | `updatedAt DateTime @updatedAt` |
| Map model to table name | `CREATE TABLE "user_accounts" (...)` | `@@map("user_accounts")` on the model |
| Map field to column name | `first_name TEXT NOT NULL` | `firstName String @map("first_name")` |
| Check existing tables | `\dt` or `SELECT table_name FROM information_schema.tables` | `npx prisma db pull` |
| Drop table | `DROP TABLE users;` | Remove model from schema + `prisma migrate dev` |
| If not exists | `CREATE TABLE IF NOT EXISTS users (...)` | Prisma Migrate handles idempotency automatically |

### Code: SQL — full unified schema

```sql
CREATE TYPE user_role AS ENUM ('admin', 'user', 'moderator');

CREATE TABLE users (
  id         SERIAL      PRIMARY KEY,
  name       TEXT        NOT NULL,
  email      TEXT        NOT NULL UNIQUE,
  role       user_role   NOT NULL DEFAULT 'user',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE posts (
  id         SERIAL      PRIMARY KEY,
  title      TEXT        NOT NULL,
  body       TEXT        NOT NULL DEFAULT '',
  published  BOOLEAN     NOT NULL DEFAULT false,
  author_id  INTEGER     NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE comments (
  id         SERIAL      PRIMARY KEY,
  body       TEXT        NOT NULL,
  author_id  INTEGER     NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id    INTEGER     NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE tags (
  id   SERIAL PRIMARY KEY,
  name TEXT   NOT NULL UNIQUE
);

CREATE TABLE post_tags (
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id  INTEGER NOT NULL REFERENCES tags(id)  ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);
```

### Code: Prisma — full unified schema

```prisma
enum Role {
  admin
  user
  moderator
}

model User {
  id        Int       @id @default(autoincrement())
  name      String
  email     String    @unique
  role      Role      @default(user)
  createdAt DateTime  @default(now())
  posts     Post[]
  comments  Comment[]
}

model Post {
  id        Int       @id @default(autoincrement())
  title     String
  body      String    @default("")
  published Boolean   @default(false)
  authorId  Int
  author    User      @relation(fields: [authorId], references: [id], onDelete: Cascade)
  createdAt DateTime  @default(now())
  comments  Comment[]
  tags      Tag[]
}

model Comment {
  id        Int      @id @default(autoincrement())
  body      String
  authorId  Int
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  postId    Int
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]
}
```

---

## What's Really Happening

Prisma models are **not tables** — they are a description of tables that Prisma uses to:
1. Generate TypeScript types for your application code.
2. Generate SQL migration files when you run `prisma migrate dev`.
3. Build the query API (`prisma.user.findMany()`, etc.).

**Naming conventions:** Prisma convention is PascalCase model names (`User`, `Post`) mapping to snake_case or lowercase table names in SQL. The generated table names follow the model name by default (`User` → `"User"` in PostgreSQL, quoted). Use `@@map("users")` to force lowercase snake_case table names, which is more conventional for PostgreSQL.

**`@updatedAt`** — there is no native PostgreSQL equivalent in a single column definition. In raw SQL you'd need a trigger function that fires `BEFORE UPDATE` to set `updated_at = NOW()`. Prisma handles this in the application layer: every `update()` call automatically sets the `updatedAt` field to the current timestamp before sending the SQL. This means `updatedAt` is only accurate when updates go through Prisma — raw SQL updates bypass it.

**Optional fields (`?`)** — in Prisma, a field without `?` is `NOT NULL` in SQL. A field with `?` is nullable. This maps directly, but beginners sometimes expect `?` to mean "optional to provide when creating" — it doesn't. An optional field just means the database allows NULL; whether you must supply it depends on whether there's a default.

**Relations in Prisma** — the `posts Post[]` on `User` and `author User` on `Post` are **not columns**. They are virtual relation fields that Prisma uses to understand the foreign key link. Only `authorId Int` is a real column. We'll cover relations in depth in [12 — Relationships](./12-relationships.md).

---

## Practice

1. Write the `CREATE TABLE` SQL for a `profiles` table with: an integer PK that references `users(id)`, a nullable `bio TEXT`, and an `avatar_url TEXT`. Then write the equivalent Prisma `model Profile`.

2. Prisma uses camelCase field names (`createdAt`) mapping to snake_case columns (`created_at`) via `@map`. Why does this matter? What breaks if you omit the `@map` and just name the field `created_at` in Prisma?

3. What does `@@map("posts")` do to the generated SQL? Run `npx prisma migrate dev --create-only` (don't apply) and read the generated SQL file to verify.

---

### Documentation Links

- PostgreSQL `CREATE TABLE`: [CREATE TABLE](https://www.postgresql.org/docs/current/sql-createtable.html)
- PostgreSQL `CREATE TYPE` (enum): [CREATE TYPE](https://www.postgresql.org/docs/current/sql-createtype.html)
- Prisma models reference: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/models)
- Prisma naming conventions: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/models#naming-models)
