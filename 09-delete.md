# 09. DELETE

**Goal:** Remove rows from a table — single records, bulk deletes, cascade behavior, and the soft-delete pattern.
**Prerequisites:** [08 — UPDATE](./08-update.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| Delete one row by PK | `DELETE FROM users WHERE id = 1;` | `prisma.user.delete({ where: { id: 1 } })` |
| Delete one row, return it | `DELETE FROM users WHERE id = 1 RETURNING *;` | `delete()` always returns the deleted record |
| Delete many rows | `DELETE FROM users WHERE role = 'guest';` | `prisma.user.deleteMany({ where: { role: 'guest' } })` |
| `deleteMany` return value | `DELETE ... RETURNING *` returns rows | `deleteMany` returns `{ count: n }` — not the deleted records |
| Delete all rows | `DELETE FROM users;` or `TRUNCATE users;` | `prisma.user.deleteMany()` (no `where` = all rows; no `TRUNCATE` equivalent) |
| Cascade delete (FK) | `ON DELETE CASCADE` on the foreign key column | `@relation(onDelete: Cascade)` in schema |
| Restrict delete (FK) | `ON DELETE RESTRICT` | `@relation(onDelete: Restrict)` — default in Prisma |
| Set null on delete (FK) | `ON DELETE SET NULL` | `@relation(onDelete: SetNull)` |
| Soft delete | `UPDATE users SET deleted_at = NOW() WHERE id = 1;` | `prisma.user.update({ where: { id: 1 }, data: { deletedAt: new Date() } })` — no built-in support |
| Delete if exists (no error) | `DELETE FROM users WHERE id = 1` (0 rows = success in SQL) | `delete()` throws `P2025` if record not found; use `deleteMany({ where: { id: 1 } })` to delete-if-exists |

### Code: Delete one user

```sql
-- SQL
DELETE FROM users
WHERE id = 1
RETURNING *;
```

```js
// Prisma
const deleted = await prisma.user.delete({
  where: { id: 1 }
})
// Returns the deleted User object
// Throws P2025 if user with id=1 does not exist
```

### Code: Delete many

```sql
-- SQL: delete all unpublished posts older than 90 days
DELETE FROM posts
WHERE published = false
  AND created_at < NOW() - INTERVAL '90 days';
```

```js
// Prisma
const result = await prisma.post.deleteMany({
  where: {
    published: false,
    createdAt: { lt: new Date(Date.now() - 90 * 24 * 60 * 60 * 1000) }
  }
})
// Returns: { count: n }
```

### Code: Cascade delete (schema definition)

```sql
-- SQL: when a user is deleted, their posts are automatically deleted
ALTER TABLE posts
  ADD CONSTRAINT fk_posts_author
  FOREIGN KEY (author_id) REFERENCES users(id) ON DELETE CASCADE;
```

```prisma
// Prisma schema
model Post {
  id       Int  @id @default(autoincrement())
  authorId Int
  author   User @relation(fields: [authorId], references: [id], onDelete: Cascade)
}
```

```js
// Now deleting a user also deletes all their posts
await prisma.user.delete({ where: { id: 1 } })
// SQL executed: DELETE FROM users WHERE id = 1
// Postgres CASCADE then deletes all posts with author_id = 1
```

### Code: Soft delete pattern

```sql
-- SQL: mark as deleted rather than removing
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

UPDATE users SET deleted_at = NOW() WHERE id = 1;

-- Exclude soft-deleted rows in queries
SELECT * FROM users WHERE deleted_at IS NULL;
```

```js
// Prisma: soft delete = update, not delete
await prisma.user.update({
  where: { id: 1 },
  data: { deletedAt: new Date() }
})

// Query only active users
const activeUsers = await prisma.user.findMany({
  where: { deletedAt: null }
})
```

### Code: Delete-if-exists (no error on missing)

```sql
-- SQL: deletes 0 rows if not found — no error
DELETE FROM users WHERE id = 999;
```

```js
// Prisma: deleteMany with a PK filter = delete-if-exists
const result = await prisma.user.deleteMany({
  where: { id: 999 }
})
// Returns { count: 0 } if not found — no throw
```

---

## What's Really Happening

**`delete()` throws on missing records** — unlike SQL's `DELETE WHERE id = 1` (which succeeds silently on zero rows), Prisma's `delete()` throws `P2025` if no record matches. This is often the right behavior — a missing delete is usually a bug — but sometimes you want delete-if-exists. In that case, use `deleteMany({ where: { id } })` and check the returned `count`.

**`TRUNCATE` vs `DELETE`** — `TRUNCATE` is much faster for clearing entire tables because it skips scanning rows and just deallocates pages. Prisma has no `TRUNCATE` equivalent. `deleteMany()` with no filter compiles to `DELETE FROM table` (not `TRUNCATE`). For bulk-clearing tables in a test setup, use `$executeRaw\`TRUNCATE TABLE users CASCADE\``.

**`ON DELETE CASCADE` in the database** — the cascade behavior is enforced at the PostgreSQL level, not in Prisma's application code. This means cascade deletes happen even if you delete a user via raw SQL or the Supabase dashboard, not just through Prisma. This is the correct behavior — enforce referential integrity at the database level.

**Soft delete is not a Prisma feature** — Prisma has no built-in soft delete middleware out of the box. Common approaches: (1) add `deletedAt DateTime?` manually and filter in every query, (2) use a Prisma middleware/extension that automatically adds `WHERE deleted_at IS NULL`, (3) use a database view that filters deleted rows and expose it as a Prisma `view`. All three require manual setup.

> **Leaky abstraction alert:** If you use `ON DELETE CASCADE` on FK relationships, Prisma's `delete()` may trigger cascades that delete far more rows than you expect — silently, no error, no return of what was deleted. Always check what cascades are attached to a table before deleting.

---

## Practice

1. Write a SQL `DELETE` and a Prisma `delete()` that removes a comment by its `id`. What happens if the comment doesn't exist in each case?

2. Add a `deletedAt DateTime?` field to the `Post` model to enable soft deletes. Write the Prisma query to "soft delete" a post, and the query to list all non-deleted posts.

3. Write the Prisma schema definition that causes comments to be automatically deleted when their parent post is deleted (`onDelete: Cascade`). Then write the SQL `ON DELETE CASCADE` equivalent.

---

### Documentation Links

- PostgreSQL `DELETE`: [DELETE](https://www.postgresql.org/docs/current/sql-delete.html)
- PostgreSQL `TRUNCATE`: [TRUNCATE](https://www.postgresql.org/docs/current/sql-truncate.html)
- PostgreSQL referential integrity: [Foreign Keys](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-FK)
- Prisma `delete`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#delete)
- Prisma `deleteMany`: (verify: https://www.prisma.io/docs/orm/reference/prisma-client-reference#deletemany)
- Prisma referential actions: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/relations/referential-actions)
