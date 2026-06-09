# 22. Row Level Security (RLS)

**Goal:** Understand what Row Level Security is, how Supabase uses it, and the critical difference between how the Supabase client and Prisma interact with RLS policies.
**Prerequisites:** [01 — Setup & Environment](./01-setup-and-environment.md), [16 — Transactions](./16-transactions.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Enable RLS on a table | `ALTER TABLE posts ENABLE ROW LEVEL SECURITY;` | `prisma.$executeRaw\`ALTER TABLE posts ENABLE ROW LEVEL SECURITY\`` (in migration SQL) |
| Disable RLS | `ALTER TABLE posts DISABLE ROW LEVEL SECURITY;` | Via raw execution or migration |
| Create a SELECT policy | `CREATE POLICY select_own ON posts FOR SELECT USING (author_id = current_user_id())` | No schema-level equivalent — write in Supabase SQL editor or migration file |
| Create an ALL policy | `CREATE POLICY all_own ON posts USING (author_id = (current_setting('app.user_id'))::int)` | Write in migration SQL; call `SET LOCAL app.user_id = '...'` before queries |
| List policies | `SELECT * FROM pg_policies;` | `prisma.$queryRaw\`SELECT * FROM pg_policies\`` |
| Check RLS on a table | `SELECT relrowsecurity FROM pg_class WHERE relname = 'posts';` | `prisma.$queryRaw\`SELECT relrowsecurity FROM pg_class WHERE relname = 'posts'\`` |
| Switch role for RLS | `SET LOCAL ROLE authenticated;` | `prisma.$executeRaw\`SET LOCAL ROLE authenticated\`` inside `$transaction` |
| Pass user ID to policies | `SET LOCAL app.current_user_id = '42';` | `prisma.$executeRaw\`SET LOCAL app.current_user_id = '${id}'\`` inside `$transaction` |
| Bypass RLS | `BYPASSRLS` privilege on the role | Prisma's service role has `BYPASSRLS` by default — RLS is skipped entirely |

### Code: Enable RLS and create a policy (in a migration)

```sql
-- Enable RLS
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- Users can only see their own posts
CREATE POLICY select_own_posts ON posts
  FOR SELECT
  USING (author_id = (current_setting('app.current_user_id', true))::int);

-- Users can only insert posts with their own author_id
CREATE POLICY insert_own_posts ON posts
  FOR INSERT
  WITH CHECK (author_id = (current_setting('app.current_user_id', true))::int);
```

### Code: How Supabase client respects RLS (for comparison)

```js
// Supabase JS client: PostgREST switches the role to "authenticated"
// and sets request.jwt.claims for each request — RLS fires correctly
const { data } = await supabase
  .from('posts')
  .select('*')
  // Supabase PostgREST handles role switching and JWT claims automatically
```

### Code: Prisma bypasses RLS by default

```js
// Prisma connects as the "postgres" service role (or the role in DATABASE_URL)
// The service role has BYPASSRLS privilege in Supabase
// This means: ALL rows are returned, no matter what RLS policies say

const posts = await prisma.post.findMany()
// Returns ALL posts from ALL users — RLS is bypassed
```

### Code: Making RLS work with Prisma (SET LOCAL pattern)

```js
// To enforce RLS in Prisma, wrap queries in a transaction
// and SET LOCAL the user context BEFORE the query

async function getPostsForUser(userId: number) {
  return await prisma.$transaction(async (tx) => {
    // 1. Switch to the "authenticated" role (which RLS policies check)
    await tx.$executeRaw`SET LOCAL ROLE authenticated`

    // 2. Set the user ID that policies read via current_setting()
    await tx.$executeRaw`SET LOCAL app.current_user_id = ${userId.toString()}`

    // 3. Now this query is subject to RLS policies
    return tx.$queryRaw`SELECT * FROM posts`
  })
}
```

### Code: Policy using JWT claims (Supabase auth integration)

```sql
-- Policy that works with Supabase Auth JWT claims
-- (used by PostgREST/Supabase client — not directly by Prisma)
CREATE POLICY select_own ON posts
  FOR SELECT
  USING (
    auth.uid() = author_id::uuid
    -- auth.uid() returns the user ID from the JWT token
  );
```

```js
// Prisma alternative: use app.current_user_id custom setting
// (set via SET LOCAL inside $transaction) rather than auth.uid()
```

---

## What's Really Happening

**What RLS is** — Row Level Security is a PostgreSQL feature that filters rows at the database engine level. You attach *policies* to a table: each policy is a SQL expression that must be `true` for a row to be visible or modifiable. Even a query like `SELECT * FROM posts` only returns rows where the policy passes.

**Supabase enables RLS on all tables by default** — when you create a new table in the Supabase dashboard, RLS is enabled with no policies, which means **no rows are accessible to non-service-role connections**. The Supabase JS client (via PostgREST) uses an `authenticated` or `anon` role, and policies control what those roles can see. This is Supabase's security model.

**Prisma uses the service role — and bypasses RLS** — Prisma connects using the `DATABASE_URL` you provide, which in Supabase is typically the `postgres` or service role. That role has the `BYPASSRLS` privilege, so RLS policies are completely ignored. From Prisma's perspective, all rows are always visible.

This means:
- The Supabase JS client (PostgREST) enforces RLS — correct for mobile/web clients where each user should only see their own data.
- Prisma — typically used in a backend/server — bypasses RLS and sees all data, relying on application-level access control.

**The `SET LOCAL` pattern** — if you need Prisma to respect RLS (e.g., you're building a multi-tenant system where the server must enforce RLS), you can impersonate a role inside a `$transaction` using `SET LOCAL ROLE`. `SET LOCAL` only lasts for the current transaction — when the transaction ends, the role is restored. This is the correct pattern for making RLS work with Prisma.

**`SET LOCAL` vs `SET`** — `SET LOCAL` changes the parameter only for the current transaction. `SET` (without LOCAL) changes it for the session. Always use `SET LOCAL` inside Prisma transactions to avoid polluting the connection state when it's returned to the pool.

> **Leaky abstraction alert:** The `SET LOCAL` pattern has a critical security implication: if your `$transaction` throws and is rolled back, the `SET LOCAL` is also undone. But if you accidentally use `SET` (without `LOCAL`), the role change persists on the pooled connection and could leak to a subsequent request from a *different* user. Always use `SET LOCAL` inside transactions.

---

## Practice

1. Enable RLS on the `posts` table using a Prisma `$executeRaw`. Write a policy that allows users to see only their own posts.

2. Write a Prisma function `getPostsForUser(userId)` that uses the `SET LOCAL` pattern to enforce the RLS policy you created. Test it — does it return only the correct user's posts?

3. What is the difference between how the Supabase JS client and Prisma interact with RLS? When would you use each in a backend server context?

---

### Documentation Links

- PostgreSQL Row Level Security: [Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- PostgreSQL `SET` / `SET LOCAL`: [SET](https://www.postgresql.org/docs/current/sql-set.html)
- PostgreSQL `current_setting`: [System Information Functions](https://www.postgresql.org/docs/current/functions-info.html)
- Supabase RLS guide: (verify: https://supabase.com/docs/guides/database/postgres/row-level-security)
- Supabase RLS with service role: (verify: https://supabase.com/docs/guides/database/postgres/row-level-security#with-check-option)
- Prisma + Supabase RLS: (verify: https://www.prisma.io/docs/orm/overview/databases/supabase#row-level-security)
