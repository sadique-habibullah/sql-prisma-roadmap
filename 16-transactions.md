# 16. Transactions

**Goal:** Group multiple operations into a single atomic unit — all succeed or all roll back together.
**Prerequisites:** [07 — INSERT](./07-insert.md), [08 — UPDATE](./08-update.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Begin a transaction | `BEGIN;` | Start of `$transaction([...])` or `$transaction(async (tx) => { ... })` |
| Commit | `COMMIT;` | End of `$transaction` block (implicit) |
| Rollback | `ROLLBACK;` | Automatic if any operation throws |
| Array transaction (batch) | `BEGIN; op1; op2; COMMIT;` | `prisma.$transaction([op1, op2])` |
| Interactive transaction | `BEGIN; conditional logic; COMMIT;` | `prisma.$transaction(async (tx) => { ... })` |
| Savepoint | `SAVEPOINT sp1;` | No Prisma equivalent |
| Rollback to savepoint | `ROLLBACK TO sp1;` | No Prisma equivalent |
| Isolation level: Read Committed | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` | `prisma.$transaction(..., { isolationLevel: 'ReadCommitted' })` |
| Isolation level: Repeatable Read | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;` | `isolationLevel: 'RepeatableRead'` |
| Isolation level: Serializable | `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;` | `isolationLevel: 'Serializable'` |
| Transaction timeout | `SET statement_timeout = '5s';` | `prisma.$transaction(..., { timeout: 5000 })` |
| Transaction in Supabase pooler | Use `DIRECT_URL` for long transactions | Transaction-mode pooling releases connection after each statement; `$transaction` holds connection for its duration — may conflict with pooler |

### Code: Array transaction (all-or-nothing batch)

```sql
-- SQL
BEGIN;
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
INSERT INTO posts (title, author_id) VALUES ('Hello', 1);
COMMIT;
-- If either INSERT fails, both are rolled back
```

```js
// Prisma: array form — all operations run in one transaction
const [user, post] = await prisma.$transaction([
  prisma.user.create({ data: { name: 'Alice', email: 'alice@example.com' } }),
  prisma.post.create({ data: { title: 'Hello', authorId: 1 } })
])
// If either throws, both are rolled back
```

### Code: Interactive transaction (with conditional logic)

```sql
-- SQL
BEGIN;
  SELECT balance FROM wallets WHERE user_id = 1 FOR UPDATE;
  -- Check balance in application, then:
  UPDATE wallets SET balance = balance - 100 WHERE user_id = 1;
  UPDATE wallets SET balance = balance + 100 WHERE user_id = 2;
COMMIT;
```

```js
// Prisma: interactive (callback) form — use tx instead of prisma
await prisma.$transaction(async (tx) => {
  const sender = await tx.wallet.findUnique({
    where: { userId: 1 }
  })

  if (sender.balance < 100) {
    throw new Error('Insufficient funds')  // causes rollback
  }

  await tx.wallet.update({
    where: { userId: 1 },
    data: { balance: { decrement: 100 } }
  })

  await tx.wallet.update({
    where: { userId: 2 },
    data: { balance: { increment: 100 } }
  })
})
```

### Code: Isolation level

```sql
-- SQL: prevent phantom reads
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SELECT COUNT(*) FROM posts WHERE author_id = 5;
  -- ... other reads ...
COMMIT;
```

```js
// Prisma
await prisma.$transaction(
  async (tx) => {
    const count1 = await tx.post.count({ where: { authorId: 5 } })
    // ... other reads — snapshot is consistent within this transaction
  },
  { isolationLevel: 'RepeatableRead' }
)
```

### Code: Supabase pooling + transactions

```js
// IMPORTANT: When using Supabase's Supavisor in transaction mode,
// $transaction holds a real database connection for its entire duration.
// Ensure your DATABASE_URL uses ?pgbouncer=true to disable prepared statements.
// For long-running transactions, consider using DIRECT_URL.

// Short transactions (< 500ms) work fine through the pooler
await prisma.$transaction([
  prisma.post.create({ data: { title: 'Quick post', authorId: 1 } }),
  prisma.user.update({ where: { id: 1 }, data: { name: 'Updated' } })
])
```

---

## What's Really Happening

**Atomicity** — in a transaction, either ALL operations succeed and are permanently written to the database, or NONE of them are. There is no partial state. This is essential for operations that must stay consistent — like transferring money (debit + credit must both succeed or neither should).

**Array vs interactive transactions:**
- **Array form** `prisma.$transaction([...])` — Prisma sends all operations to the database together in a single transaction. The operations are pre-built before the transaction starts. This is efficient but you can't read results mid-transaction to make decisions.
- **Interactive form** `prisma.$transaction(async (tx) => { ... })` — opens a real SQL `BEGIN`, lets you read + write + branch on results, then commits or rolls back. Any thrown error (including your own `throw new Error(...)`) causes a `ROLLBACK`. Use `tx` (not `prisma`) for all calls inside this block.

**Supabase Supavisor and transactions** — Supabase's connection pooler in **transaction mode** releases the database connection back to the pool after each individual statement. Prisma's `$transaction` holds a connection for the whole block, which can cause issues if the pooler reassigns connections between statements. Mitigation: keep transactions short, and for critical long transactions consider using the `DIRECT_URL` or switching Supavisor to session mode.

**The `timeout` option** — by default Prisma's interactive transactions time out after 5 seconds. Increase this for complex operations: `{ timeout: 30000 }` (30 seconds). Long-running transactions hold a database connection and lock rows — keep them as short as possible.

> **Leaky abstraction alert:** In the array form, you can't use the result of operation 1 as input for operation 2 (e.g., insert user, then use the returned `id` in the next insert). For that, you must use the interactive form or Prisma's nested write API. The array form is best for independent operations that happen to need the same atomicity guarantee.

---

## Practice

1. Write a SQL `BEGIN`/`COMMIT` block and Prisma `$transaction` (interactive form) that creates a new post and simultaneously increments the author's `post_count` field.

2. When would you choose the array form over the interactive form? Write one example of each and explain the tradeoff.

3. A transfer function moves funds from wallet A to wallet B. Write it using Prisma's interactive transaction. What happens if wallet A doesn't have enough funds? Make it throw and roll back.

---

### Documentation Links

- PostgreSQL transactions: [Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)
- PostgreSQL isolation levels: [Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- Prisma `$transaction`: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/transactions)
- Prisma interactive transactions: (verify: https://www.prisma.io/docs/orm/prisma-client/queries/transactions#interactive-transactions)
- Supabase connection pooling modes: (verify: https://supabase.com/docs/guides/database/connecting-to-postgres#connection-pooler)
