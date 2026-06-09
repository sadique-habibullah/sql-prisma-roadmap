# 03. Data Types

**Goal:** Map every common PostgreSQL type to its Prisma scalar type and understand where the mapping is imprecise.
**Prerequisites:** [02 — Databases, Schemas & Connections](./02-databases-schemas-connections.md)

---

## Comparison Table

| PostgreSQL Type | Example SQL | Prisma Type | Notes |
|---|---|---|---|
| `SERIAL` | `id SERIAL PRIMARY KEY` | `Int @id @default(autoincrement())` | `SERIAL` is shorthand for an integer + sequence |
| `BIGSERIAL` | `id BIGSERIAL PRIMARY KEY` | `BigInt @id @default(autoincrement())` | JS receives a `bigint` primitive, not `number` |
| `INTEGER` / `INT4` | `age INTEGER` | `Int` | |
| `BIGINT` / `INT8` | `views BIGINT` | `BigInt` | Requires `BigInt` handling in JSON responses |
| `SMALLINT` / `INT2` | `score SMALLINT` | `Int` | Prisma has no `SmallInt`; stored as `int2` in DB |
| `REAL` / `FLOAT4` | `ratio REAL` | `Float` | Lossy floating-point |
| `DOUBLE PRECISION` / `FLOAT8` | `price DOUBLE PRECISION` | `Float` | |
| `NUMERIC` / `DECIMAL` | `amount NUMERIC(10,2)` | `Decimal` | Returns `Decimal.js` object, not `number` — use for money |
| `TEXT` | `bio TEXT` | `String` | Unlimited length |
| `VARCHAR(n)` | `name VARCHAR(100)` | `String @db.VarChar(100)` | `@db` attribute required for length constraint |
| `CHAR(n)` | `code CHAR(3)` | `String @db.Char(3)` | Pads with spaces to fixed length |
| `BOOLEAN` | `published BOOLEAN` | `Boolean` | |
| `DATE` | `birthday DATE` | `DateTime @db.Date` | Date only — no time component |
| `TIMESTAMP` | `updated_at TIMESTAMP` | `DateTime @db.Timestamp(n)` | No timezone — avoid in new tables |
| `TIMESTAMPTZ` | `created_at TIMESTAMPTZ DEFAULT NOW()` | `DateTime` | Prisma's default `DateTime` maps to `TIMESTAMPTZ` |
| `UUID` | `id UUID DEFAULT gen_random_uuid()` | `String @id @default(uuid()) @db.Uuid` | Prisma generates UUID client-side with `uuid()`; `@db.Uuid` stores as native UUID |
| `JSONB` | `metadata JSONB` | `Json` | JSONB (binary JSON) — prefer over `JSON` |
| `JSON` | `data JSON` | `Json @db.Json` | Stored as text, slower to query |
| `TEXT[]` | `tags TEXT[]` | `String[]` | Scalar lists require PostgreSQL provider |
| `INTEGER[]` | `scores INTEGER[]` | `Int[]` | |
| `BYTEA` | `avatar BYTEA` | `Bytes` | Returns `Buffer` in Node.js |
| `INET` | `ip_address INET` | `String @db.Inet` | No Prisma scalar for IP; stored as text |
| `CIDR` | `network CIDR` | `String` | |
| `INTERVAL` | `duration INTERVAL` | Not supported | Use `$queryRaw` or store as seconds in `Int` |
| Custom `ENUM` | `CREATE TYPE role AS ENUM ('admin', 'user')` | `enum Role { admin user }` | Prisma creates the Postgres enum type in migrations |

### Code: Prisma model using diverse types

```prisma
model Product {
  id          String   @id @default(uuid()) @db.Uuid
  name        String   @db.VarChar(200)
  price       Decimal  @db.Decimal(10, 2)
  stock       Int
  isAvailable Boolean  @default(true)
  metadata    Json?
  tags        String[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

### Code: Equivalent SQL

```sql
CREATE TABLE "Product" (
  id            UUID            NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
  name          VARCHAR(200)    NOT NULL,
  price         DECIMAL(10, 2)  NOT NULL,
  stock         INTEGER         NOT NULL,
  "isAvailable" BOOLEAN         NOT NULL DEFAULT true,
  metadata      JSONB,
  tags          TEXT[]          NOT NULL DEFAULT '{}',
  "createdAt"   TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
  "updatedAt"   TIMESTAMPTZ     NOT NULL
);
```

---

## What's Really Happening

Prisma's type system maps to PostgreSQL types at the column level, but the JavaScript/TypeScript types you work with are different from what's stored. Key gotchas:

**`BigInt`** — PostgreSQL `BIGINT` can hold numbers larger than JavaScript's `Number.MAX_SAFE_INTEGER` (2^53 - 1). Prisma correctly returns a JS `bigint`, but `JSON.stringify` does not serialize `bigint` by default. You must convert to `string` before sending a `BigInt` over HTTP.

**`Decimal`** — PostgreSQL `NUMERIC` is exact (no floating-point error). Prisma returns a `Decimal.js` object, not a native JS `number`. Never compare with `===` or do arithmetic with `+` — use the `Decimal.js` API: `.equals()`, `.add()`, `.toNumber()`.

**`DateTime`** — Prisma always stores `DateTime` as `TIMESTAMPTZ` (with timezone) unless you add `@db.Timestamp` or `@db.Date`. Postgres stores `TIMESTAMPTZ` in UTC internally. Prisma returns a JS `Date` object.

**`@db.*` attributes** — these let you override Prisma's default type mapping. Without `@db.VarChar(100)`, Prisma generates a `TEXT` column. The `@db` attribute is purely a hint to the migration generator; it does not affect the Prisma Client API.

**`SERIAL` vs `autoincrement()`** — Prisma's `@default(autoincrement())` generates a `SERIAL` column in migrations (or a `SEQUENCE` + `DEFAULT nextval(...)` for `BIGINT`). At the SQL level these are equivalent.

> **Leaky abstraction alert:** Prisma has no equivalent for PostgreSQL's `INTERVAL`, `POINT`, `POLYGON`, geometric types, or range types (`int4range`, `tstzrange`). For columns using these types, use `Unsupported("interval")` in the schema and handle them via `$queryRaw`.

---

## Practice

1. Write a Prisma model for an `orders` table with: a UUID primary key, a `Decimal` amount, a `Boolean` for paid status, and a `TIMESTAMPTZ` created-at timestamp. Then write the equivalent `CREATE TABLE` SQL.

2. Add a column `scores INTEGER[]` to a model. What Prisma type do you use? What SQL does `prisma migrate dev` generate for it?

3. Create a Prisma `enum` called `Status` with values `active`, `inactive`, `pending`. What SQL does Prisma generate for this enum type?

---

### Documentation Links

- PostgreSQL data types reference: [Data Types](https://www.postgresql.org/docs/current/datatype.html)
- PostgreSQL numeric types: [Numeric Types](https://www.postgresql.org/docs/current/datatype-numeric.html)
- PostgreSQL date/time types: [Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html)
- Prisma scalar types: (verify: https://www.prisma.io/docs/orm/reference/prisma-schema-reference#model-field-scalar-types)
- Prisma `@db` native type attributes: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/models#native-types-mapping)
