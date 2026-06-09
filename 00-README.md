# SQL ↔ Prisma Learning Roadmap (Supabase)

> **The core mental model: SQL is the core, Prisma is the shell.**
>
> Every concept is taught SQL-first. You see the real `SELECT`, `JOIN`, and `CREATE TABLE` before you see what Prisma generates. Prisma is a convenience layer — a skilled developer knows what it's doing underneath, recognizes when it's hiding something important, and knows when to reach past it for raw SQL.

---

## Prerequisites

Before starting, you need:
- **Node.js 18+** and npm
- **A Supabase project** — create one at [supabase.com](https://supabase.com) (free tier is fine)
- **psql** or the Supabase SQL editor (for running raw SQL exercises)
- Basic comfort with the terminal

No prior SQL or Prisma experience is required.

---

## How to Use This Roadmap

1. **Read the files in number order.** Each file builds on the previous ones.
2. **Do the Practice exercises.** Read-only learning sticks poorly. Write the SQL yourself, run it, then write the Prisma equivalent and compare.
3. **Run SQL in the Supabase SQL editor** (or psql). Run Prisma code in your Node.js project.
4. **Read "What's Really Happening"** in each file before moving on — that's where the important nuance lives.
5. **When in doubt, write the SQL first.** If you can express a query in SQL, you can always fall back to `$queryRaw`. Prisma's API is just a shortcut to that SQL.

---

## The Unified Schema

All examples use the same schema. Learn it now — you'll see it in every file.

```
users     (id, name, email, role, created_at)
posts     (id, title, body, published, author_id → users.id, created_at)
comments  (id, body, author_id → users.id, post_id → posts.id, created_at)
tags      (id, name)
post_tags (post_id → posts.id, tag_id → tags.id)   -- many-to-many
```

The full SQL `CREATE TABLE` statements and Prisma `model` definitions are in [04 — Tables & Models](./04-tables-and-models.md).

---

## Recommended Order

### Foundation (start here)
| # | File | What You'll Learn |
|---|---|---|
| 01 | [Setup & Environment](./01-setup-and-environment.md) | Install Prisma, connect to Supabase, understand the two connection URLs |
| 02 | [Databases, Schemas & Connections](./02-databases-schemas-connections.md) | How Postgres organizes objects; Supabase reserved schemas |
| 03 | [Data Types](./03-data-types.md) | Every PostgreSQL type mapped to its Prisma equivalent |
| 04 | [Tables & Models](./04-tables-and-models.md) | `CREATE TABLE` ↔ Prisma `model`; introduces the unified schema |

### CRUD Operations
| # | File | What You'll Learn |
|---|---|---|
| 05 | [SELECT Basics](./05-select-basics.md) | `SELECT *`, `findMany`, `findUnique`, `findFirst`, `count` |
| 06 | [Filtering with WHERE](./06-filtering-where.md) | `WHERE` operators ↔ Prisma filter operators |
| 07 | [INSERT](./07-insert.md) | `INSERT` ↔ `create`, `createMany`, nested creates, `upsert` |
| 08 | [UPDATE](./08-update.md) | `UPDATE` ↔ `update`, `updateMany`, atomic increments |
| 09 | [DELETE](./09-delete.md) | `DELETE` ↔ `delete`, `deleteMany`, cascade, soft delete |
| 10 | [Sorting & Pagination](./10-sorting-pagination.md) | `ORDER BY`, `LIMIT`, `OFFSET` ↔ `orderBy`, `take`, `skip`, cursor pagination |

### Schema Design & Intermediate Queries
| # | File | What You'll Learn |
|---|---|---|
| 11 | [Constraints & Defaults](./11-constraints-defaults.md) | `PRIMARY KEY`, `UNIQUE`, `CHECK`, `DEFAULT` ↔ Prisma attributes |
| 12 | [Relationships](./12-relationships.md) | Foreign keys, 1-1, 1-many, many-many ↔ Prisma relations |
| 13 | [JOINs vs Includes](./13-joins-vs-includes.md) | SQL `JOIN` ↔ Prisma `include`; what Prisma actually generates |
| 14 | [Aggregations & Grouping](./14-aggregations-grouping.md) | `COUNT/SUM/AVG`, `GROUP BY`, `HAVING` ↔ `aggregate`, `groupBy` |
| 15 | [Indexes](./15-indexes.md) | `CREATE INDEX`, partial and GIN indexes ↔ Prisma `@@index` |
| 16 | [Transactions](./16-transactions.md) | `BEGIN/COMMIT/ROLLBACK` ↔ `$transaction` (array and interactive) |

### Advanced SQL & Data Modeling
| # | File | What You'll Learn |
|---|---|---|
| 17 | [JSON, Arrays & Enums](./17-json-arrays-enums.md) | `JSONB`, array columns, enum types ↔ Prisma equivalents |
| 18 | [Views](./18-views.md) | `CREATE VIEW`, materialized views ↔ Prisma `view` |
| 19 | [Subqueries & CTEs](./19-subqueries-and-ctes.md) | Subqueries, `WITH` CTEs ↔ what Prisma can/can't express |
| 20 | [Raw Queries](./20-raw-queries.md) | `$queryRaw`, `$executeRaw`, `Prisma.sql`, safe parameterization |

### Supabase-Specific & Operational
| # | File | What You'll Learn |
|---|---|---|
| 21 | [Migrations](./21-migrations.md) | `prisma migrate dev/deploy`, Supabase coexistence, single source of truth |
| 22 | [Row Level Security (RLS)](./22-supabase-rls.md) | Supabase RLS, how Prisma bypasses it, the `SET LOCAL` pattern |
| 23 | [Performance & Pooling](./23-performance-and-pooling.md) | `EXPLAIN/ANALYZE`, N+1 problem, Supavisor + `pgbouncer=true` |
| 24 | [Full-Text Search](./24-full-text-search.md) | `tsvector`/`tsquery`, GIN indexes, vs Prisma's limited FTS preview |

---

## Key Themes Across the Roadmap

### Where Prisma shines
- CRUD operations: `create`, `findMany`, `update`, `delete` cover 80% of application queries
- Type safety: TypeScript types are generated from your schema
- Nested writes: create a user and their posts in one atomic call
- Relation filters: `some`, `none`, `every` replace many subqueries

### Where you must reach for raw SQL
- CTEs and recursive queries → [19 — Subqueries & CTEs](./19-subqueries-and-ctes.md)
- Window functions (`ROW_NUMBER`, `RANK`, `LAG`) → [14 — Aggregations](./14-aggregations-grouping.md)
- Partial indexes and GIN indexes → [15 — Indexes](./15-indexes.md)
- `CHECK` constraints → [11 — Constraints](./11-constraints-defaults.md)
- Full-text search at scale → [24 — Full-Text Search](./24-full-text-search.md)
- Row Level Security policies → [22 — RLS](./22-supabase-rls.md)

### Supabase specifics
- **Two connection URLs** — always required: see [01 — Setup](./01-setup-and-environment.md)
- **Reserved schemas** — never let Prisma touch `auth`, `storage`, `realtime`: see [02 — Schemas](./02-databases-schemas-connections.md)
- **Migrations as source of truth** — one approach, consistently: see [21 — Migrations](./21-migrations.md)
- **RLS and the service role** — Prisma bypasses RLS by default: see [22 — RLS](./22-supabase-rls.md)
- **`pgbouncer=true` and connection limits** — required for serverless: see [23 — Performance](./23-performance-and-pooling.md)

---

*Good luck. When a query doesn't work or Prisma can't express what you need — that's not a failure, it's the lesson. Write the SQL, understand it, and you'll always know what to do.*
