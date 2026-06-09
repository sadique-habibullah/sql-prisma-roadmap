# 01. Setup & Environment

**Goal:** Install Prisma, connect it to a Supabase Postgres database using both connection strings, and understand why two URLs are needed.
**Prerequisites:** A Supabase project with a database. Node.js 18+.

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Install client library | `npm install pg` | `npm install prisma @prisma/client` |
| Initialize project | Write connection string in code | `npx prisma init` (creates `prisma/schema.prisma` + `.env`) |
| Direct connection (port 5432) | `postgresql://postgres:[password]@db.[ref].supabase.co:5432/postgres` | `DIRECT_URL` in `.env` — used for migrations |
| Pooled connection (port 6543) | `postgresql://postgres.[ref]:[password]@aws-0-[region].pooler.supabase.com:6543/postgres` | `DATABASE_URL` in `.env` — used for all runtime queries |
| `datasource` block | N/A | `datasource db { provider url directUrl }` in `schema.prisma` |
| `generator` block | N/A | `generator client { provider = "prisma-client-js" }` in `schema.prisma` |
| Generate client | N/A | `npx prisma generate` |
| Test connection | `psql $DATABASE_URL -c "SELECT version()"` | `npx prisma db pull` (introspects schema) |

### Code: `.env`

```env
# Pooled connection — runtime queries (Supavisor, port 6543)
DATABASE_URL="postgresql://postgres.xxxx:password@aws-0-us-east-1.pooler.supabase.com:6543/postgres?pgbouncer=true&connection_limit=1"

# Direct connection — migrations only (port 5432)
DIRECT_URL="postgresql://postgres:password@db.xxxx.supabase.co:5432/postgres"
```

### Code: `prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}
```

### Code: Basic client usage

```js
// SQL equivalent: SELECT 1 + 1 AS result
const result = await prisma.$queryRaw`SELECT 1 + 1 AS result`
console.log(result) // [{ result: 2n }]
```

---

## What's Really Happening

Supabase sits a connection pooler called **Supavisor** in front of your Postgres database. When your app connects on port **6543**, it hits the pooler. The pooler multiplexes many app connections onto a small number of real Postgres connections — essential for serverless runtimes (Vercel, AWS Lambda) where each function invocation would otherwise open a new database connection.

Prisma needs **two separate URLs** because:

- **`DATABASE_URL` (port 6543, Supavisor)** — used by `PrismaClient` at runtime. Every `prisma.user.findMany()` call goes through the pooler.
- **`DIRECT_URL` (port 5432, direct)** — used only by `prisma migrate dev` and `prisma migrate deploy`. Migrations run DDL statements (`CREATE TABLE`, `ALTER TABLE`) that must execute outside a pooled transaction context. Running migrations through the pooler will fail with cryptic errors.

The `?pgbouncer=true` query parameter on `DATABASE_URL` tells Prisma to disable **prepared statements**. Supavisor in transaction mode does not support them — each "connection" returned to the pool may go to a different backend process, so a prepared statement from connection A is invisible to connection B. With `pgbouncer=true`, Prisma switches to simple query protocol, which works correctly.

The `connection_limit=1` is recommended for serverless environments to prevent each Lambda/Edge function instance from holding more than one connection.

> **Leaky abstraction alert:** If you forget `directUrl` and try to run `prisma migrate dev`, you will get a `P3009` or timeout error on Supabase. This is one of the most common Supabase + Prisma setup mistakes.

---

## Practice

1. Create a `.env` file with both `DATABASE_URL` (port 6543) and `DIRECT_URL` (port 5432) using your Supabase project's connection strings. Run `npx prisma db pull` and confirm it succeeds.

2. Open `psql` with your `DIRECT_URL` and run `SELECT current_database(), current_schema()`. Then write the Prisma equivalent using `prisma.$queryRaw`.

3. In `schema.prisma`, what happens if you remove `directUrl` and try to run `npx prisma migrate dev`? Predict the error before trying it.

---

### Documentation Links

- PostgreSQL connection strings: [libpq connection string format](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING)
- Prisma data sources: [Data sources reference](https://www.prisma.io/docs/orm/prisma-schema/overview/data-sources)
- Supabase connection pooling: [Connecting to Postgres](https://supabase.com/docs/guides/database/connecting-to-postgres)
- Prisma + Supabase guide: (verify: https://www.prisma.io/docs/orm/overview/databases/supabase)
