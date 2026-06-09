# 12. Relationships

**Goal:** Define foreign keys and model all three cardinality types — one-to-one, one-to-many, and many-to-many — in both SQL and Prisma.
**Prerequisites:** [11 — Constraints & Defaults](./11-constraints-defaults.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| One-to-many FK | `author_id INT REFERENCES users(id)` | `authorId Int` + `author User @relation(...)` |
| One-to-one FK | FK column + `UNIQUE` on it | `@unique` on scalar FK + `@relation` on both sides |
| Many-to-many (explicit) | Explicit join table with two FK columns | Explicit join model with two `@relation` fields |
| Many-to-many (implicit) | Same explicit SQL join table | `posts Post[]` on Tag + `tags Tag[]` on Post — Prisma manages `_PostToTag` |
| Cascade delete | `ON DELETE CASCADE` | `@relation(onDelete: Cascade)` |
| Cascade update | `ON UPDATE CASCADE` | `@relation(onUpdate: Cascade)` |
| Restrict delete | `ON DELETE RESTRICT` | `@relation(onDelete: Restrict)` — Prisma default |
| Set NULL on delete | `ON DELETE SET NULL` | `@relation(onDelete: SetNull)` — FK must be nullable (`?`) |
| Self-referential | `parent_id INT REFERENCES categories(id)` | `parent Category? @relation("CategoryTree", fields: [...], references: [...])` |
| Named FK constraint | `CONSTRAINT fk_posts_author FOREIGN KEY (author_id) REFERENCES users(id)` | Prisma names constraints automatically |

### Code: One-to-many (User → Posts)

```sql
-- SQL: user has many posts
CREATE TABLE posts (
  id        SERIAL  PRIMARY KEY,
  title     TEXT    NOT NULL,
  author_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE
);
```

```prisma
// Prisma: Post side has the FK scalar field + relation field
model Post {
  id       Int    @id @default(autoincrement())
  title    String
  authorId Int
  author   User   @relation(fields: [authorId], references: [id], onDelete: Cascade)
}

// User side has the back-reference (not a real column)
model User {
  id    Int    @id @default(autoincrement())
  posts Post[]  // virtual — no column in the database
}
```

### Code: One-to-one (User → Profile)

```sql
-- SQL: each user has at most one profile
CREATE TABLE profiles (
  id      SERIAL  PRIMARY KEY,
  bio     TEXT,
  user_id INTEGER NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE
);
```

```prisma
// Prisma
model Profile {
  id     Int    @id @default(autoincrement())
  bio    String?
  userId Int    @unique           // @unique enforces the 1-1 constraint
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model User {
  id      Int      @id @default(autoincrement())
  profile Profile?  // optional back-reference (no column)
}
```

### Code: Many-to-many — implicit (Prisma-managed join table)

```sql
-- SQL: Prisma generates this table automatically
CREATE TABLE "_PostToTag" (
  "A" INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  "B" INTEGER NOT NULL REFERENCES tags(id)  ON DELETE CASCADE,
  PRIMARY KEY ("A", "B")
);
```

```prisma
// Prisma: just declare the arrays on both models
model Post {
  id   Int   @id @default(autoincrement())
  tags Tag[]  // Prisma infers the join table
}

model Tag {
  id    Int    @id @default(autoincrement())
  posts Post[] // Prisma infers the join table
}
```

### Code: Many-to-many — explicit join model (with extra fields)

```sql
-- SQL: explicit join table with extra fields
CREATE TABLE post_tags (
  post_id    INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id     INTEGER NOT NULL REFERENCES tags(id)  ON DELETE CASCADE,
  tagged_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (post_id, tag_id)
);
```

```prisma
// Prisma: explicit join model (required when you need extra fields)
model PostTag {
  postId   Int
  tagId    Int
  taggedAt DateTime @default(now())
  post     Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag      Tag      @relation(fields: [tagId],  references: [id], onDelete: Cascade)
  @@id([postId, tagId])
}

model Post {
  id       Int       @id @default(autoincrement())
  postTags PostTag[]
}

model Tag {
  id       Int       @id @default(autoincrement())
  postTags PostTag[]
}
```

### Code: Self-referential (category tree)

```sql
-- SQL: category can have a parent category
CREATE TABLE categories (
  id        SERIAL  PRIMARY KEY,
  name      TEXT    NOT NULL,
  parent_id INTEGER REFERENCES categories(id)
);
```

```prisma
// Prisma
model Category {
  id        Int        @id @default(autoincrement())
  name      String
  parentId  Int?
  parent    Category?  @relation("CategoryTree", fields: [parentId], references: [id])
  children  Category[] @relation("CategoryTree")
}
```

---

## What's Really Happening

**Relation fields are not columns** — in Prisma, `author User` on a `Post` model and `posts Post[]` on a `User` model are *virtual*. They don't exist as columns in the database. Only the scalar FK field (`authorId Int`) is a real column. Prisma uses the relation fields to understand how to JOIN or subquery when you use `include` or nested filters.

**Implicit vs explicit many-to-many** — use the implicit form (just `Post[]` and `Tag[]`) when you only need the IDs on both sides. Use an explicit join model the moment you need any extra data on the relationship (like `taggedAt`, an `order` field, or a `weight`). You cannot add fields to an implicit join table later without migrating to an explicit model.

**`onDelete` default is `Restrict`** — Prisma defaults FK relationships to `onDelete: Restrict`, which means you cannot delete a parent record if any child records reference it. To delete a user with posts, you must either delete their posts first, or use `onDelete: Cascade`. This is a safe default that prevents accidental orphan records.

**Self-referential relations** need a name string (the `"CategoryTree"` in both `@relation` calls) to disambiguate when a model has more than one relation to itself.

> **Leaky abstraction alert:** Prisma's implicit many-to-many join table is named `_ModelAToModelB` and uses column names `A` and `B` — these are Prisma internals. If another part of your stack (raw SQL, Supabase dashboard) queries this table, use these exact names. You cannot rename the implicit table; if you need a custom name, use an explicit join model.

---

## Practice

1. Write the SQL `CREATE TABLE` and Prisma model for a `follows` relationship where a user can follow many other users (self-referential many-to-many). Hint: you need an explicit join model.

2. Create a `Profile` model with a one-to-one relation to `User`. Write the Prisma schema and the SQL. What constraint enforces the "one" in one-to-one?

3. Convert the implicit `Post ↔ Tag` many-to-many to an explicit `PostTag` join model that adds a `createdAt` timestamp. What migration would Prisma need to run?

---

### Documentation Links

- PostgreSQL foreign keys: [Foreign Keys](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-FK)
- PostgreSQL referential actions: [Referential Integrity](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-FK)
- Prisma relations overview: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/relations)
- Prisma one-to-one relations: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/relations/one-to-one-relations)
- Prisma many-to-many relations: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/relations/many-to-many-relations)
- Prisma referential actions: (verify: https://www.prisma.io/docs/orm/prisma-schema/data-model/relations/referential-actions)
