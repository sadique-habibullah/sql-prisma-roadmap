# 02. Databases, Schemas & Connections

**Goal:** Understand how Postgres organizes objects into databases and schemas, and learn which schemas Supabase owns so you can keep Prisma out of them.
**Prerequisites:** [01 — Setup & Environment](./01-setup-and-environment.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| List all databases | `SELECT datname FROM pg_database;` | No equivalent — one `PrismaClient` instance per database |
| Create a schema | `CREATE SCHEMA myapp;` | `@@schema("myapp")` on models (requires `multiSchema` preview) |
| List schemas | `SELECT schema_name FROM information_schema.schemata;` | `npx prisma db pull` introspects configured schemas |
| Set default schema for session | `SET search_path = myapp, public;` | `datasource db { schemas = ["myapp", "public"] }` |
| List tables in a schema | `\dt myapp.*` or `SELECT table_name FROM information_schema.tables WHERE table_schema = 'myapp';` | `npx prisma db pull` (reflects whatever `schemas` array says) |
| The `public` schema | Default schema for user-created objects | Prisma default — no `@@schema` annotation needed |
| Supabase reserved schemas | `auth`, `storage`, `realtime`, `supabase_functions`, `extensions` | Exclude from `datasource.schemas`; never annotate models with these |
| Cross-schema reference | `SELECT * FROM auth.users JOIN public.profiles ON ...` | Not natively supported without `multiSchema`; use `$queryRaw` |
| Drop schema | `DROP SCHEMA myapp CASCADE;` | Remove from `schemas` array + run `prisma migrate dev` |

### Code: Inspecting schemas in Supabase

```sql
-- See every schema and who owns it
SELECT schema_name, schema_owner
FROM information_schema.schemata
ORDER BY schema_name;
```

### Code: `schema.prisma` — restricting to `public` only

```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
  // Without this, prisma db pull tries to introspect ALL schemas
  schemas   = ["public"]
}
```

### Code: Enabling `multiSchema` for two app-owned schemas

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["multiSchema"]
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
  schemas   = ["public", "analytics"]   // only schemas YOU own
}

model Event {
  id        Int    @id @default(autoincrement())
  name      String
  @@schema("analytics")
}
```

---

## What's Really Happening

A **database** is the top-level isolation unit in Postgres — objects in different databases cannot reference each other directly. Supabase gives you one database per project. Within that database, **schemas** act as namespaces, similar to folders.

Supabase creates several schemas at project setup that it controls:

| Schema | Purpose |
|---|---|
| `public` | Your application tables — the one you work in |
| `auth` | Supabase Auth: `users`, `sessions`, `identities`, etc. |
| `storage` | Supabase Storage: `buckets`, `objects` |
| `realtime` | Supabase Realtime internals |
| `extensions` | Postgres extensions (uuid-ossp, pgcrypto, etc.) |
| `supabase_functions` | Edge Function hooks |

If you run `npx prisma db pull` without specifying `schemas`, Prisma will introspect all of these and generate hundreds of models you don't own — polluting your schema file and potentially causing migration conflicts. **Always restrict `schemas` to only the namespaces you control.**

The `search_path` setting tells Postgres which schemas to check first when you reference an unqualified table name (e.g. `SELECT * FROM users` instead of `SELECT * FROM public.users`). Prisma always uses fully qualified names in the SQL it generates, so `search_path` mostly matters when writing raw SQL yourself.

> **Leaky abstraction alert:** If you reference `auth.users` in a raw query from Prisma, it works — but if you try to model it as a Prisma relation, you'll need `multiSchema` enabled and you'll be telling Prisma to manage a schema it shouldn't touch. A safer pattern is to use `auth.users` as a foreign key target in SQL but keep that relationship read-only via `$queryRaw`.

---

## Practice

1. In your Supabase SQL editor, run the query to list all schemas. Identify which are Supabase-managed and which you own.

2. Add `schemas = ["public"]` to your `datasource` block if it isn't there. Run `npx prisma db pull`. How does the generated schema compare to a pull without that restriction?

3. Write a raw SQL query that joins `auth.users` (Supabase's user table) to a hypothetical `public.profiles` table on the user's `id`. Why would you use `$queryRaw` for this rather than a Prisma relation?

---

### Documentation Links

- PostgreSQL schemas: [DDL Schemas](https://www.postgresql.org/docs/current/ddl-schemas.html)
- PostgreSQL `search_path`: [Schema Search Path](https://www.postgresql.org/docs/current/ddl-schemas.html#DDL-SCHEMAS-PATH)
- Prisma multiSchema: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/multi-schema)
- Supabase database schemas: (verify: https://supabase.com/docs/guides/database/managing-schemas)
