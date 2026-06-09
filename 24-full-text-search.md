# 24. Full-Text Search

**Goal:** Implement text search using PostgreSQL's native full-text search engine, and understand Prisma's limited built-in support vs. the raw SQL approach.
**Prerequisites:** [15 — Indexes](./15-indexes.md), [20 — Raw Queries](./20-raw-queries.md)

---

## Comparison Table

| Concept | SQL (PostgreSQL) | Prisma |
|---|---|---|
| `tsvector` type | `fts TSVECTOR` column | `fts Unsupported("tsvector")?` (no native scalar) |
| Generated `tsvector` column | `GENERATED ALWAYS AS (to_tsvector('english', title \|\| ' ' \|\| body)) STORED` | Add via migration SQL; mark field as `Unsupported` in schema |
| `to_tsvector` function | `to_tsvector('english', 'Hello world')` | No equivalent — use in `$queryRaw` |
| `plainto_tsquery` | `plainto_tsquery('english', 'hello world')` | No equivalent — use in `$queryRaw` |
| `websearch_to_tsquery` | `websearch_to_tsquery('english', 'hello -world')` | No equivalent — use in `$queryRaw` |
| FTS match operator | `WHERE fts @@ plainto_tsquery('english', $1)` | `prisma.$queryRaw` with the SQL directly |
| GIN index on `tsvector` | `CREATE INDEX ON posts USING GIN(fts)` | Add in migration SQL; no Prisma schema equivalent |
| Rank results | `ts_rank(fts, query)` in `ORDER BY` | `$queryRaw` with `ORDER BY ts_rank(...)` |
| Highlight matches | `ts_headline('english', title, query)` | `$queryRaw` — returns highlighted HTML/text snippets |
| Prisma FTS (built-in, PostgreSQL) | N/A | `where: { title: { search: 'hello & world' } }` with `fullTextSearch` preview feature |
| Prisma `@@fulltext` index | N/A | Only for MySQL — no PostgreSQL equivalent |

### Code: Adding a `tsvector` column via migration

```sql
-- Add a generated tsvector column to posts
ALTER TABLE posts
  ADD COLUMN fts TSVECTOR
  GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(body, ''))
  ) STORED;

-- Create a GIN index for fast FTS
CREATE INDEX posts_fts_idx ON posts USING GIN(fts);
```

```prisma
// In schema.prisma — mark the column as Unsupported
// so Prisma knows it exists but won't try to manage it
model Post {
  id    Int                     @id @default(autoincrement())
  title String
  body  String
  fts   Unsupported("tsvector")? // read-only; populated by Postgres

  @@index([fts], type: Brin)    // Note: use raw migration for GIN index
}
```

### Code: Full-text search query via `$queryRaw`

```sql
-- SQL: find posts matching 'prisma tutorial'
SELECT
  id,
  title,
  ts_rank(fts, query) AS rank
FROM posts,
     plainto_tsquery('english', 'prisma tutorial') AS query
WHERE fts @@ query
ORDER BY rank DESC
LIMIT 10;
```

```js
// Prisma: $queryRaw with parameterized query
type SearchResult = { id: number; title: string; rank: number }

async function searchPosts(searchTerm: string, limit = 10): Promise<SearchResult[]> {
  return prisma.$queryRaw<SearchResult[]>`
    SELECT
      id,
      title,
      ts_rank(fts, query) AS rank
    FROM posts,
         plainto_tsquery('english', ${searchTerm}) AS query
    WHERE fts @@ query
    ORDER BY rank DESC
    LIMIT ${limit}
  `
}
```

### Code: Web-style search (with AND, OR, negation)

```sql
-- websearch_to_tsquery supports natural syntax:
-- 'prisma OR supabase' → prisma | supabase
-- '-old tutorial'      → !old & tutorial
-- '"exact phrase"'     → exact <-> phrase (adjacent)

SELECT title FROM posts
WHERE fts @@ websearch_to_tsquery('english', 'prisma OR supabase -old');
```

```js
const posts = await prisma.$queryRaw`
  SELECT id, title
  FROM posts
  WHERE fts @@ websearch_to_tsquery('english', ${searchInput})
  ORDER BY ts_rank(fts, websearch_to_tsquery('english', ${searchInput})) DESC
`
```

### Code: Highlighted matches

```sql
-- ts_headline returns the original text with matching words highlighted
SELECT
  title,
  ts_headline(
    'english',
    body,
    plainto_tsquery('english', 'prisma'),
    'MaxWords=50, MinWords=10, StartSel=<b>, StopSel=</b>'
  ) AS excerpt
FROM posts
WHERE fts @@ plainto_tsquery('english', 'prisma');
```

```js
const results = await prisma.$queryRaw`
  SELECT
    title,
    ts_headline(
      'english',
      body,
      plainto_tsquery('english', ${term}),
      'MaxWords=50, MinWords=10, StartSel=<b>, StopSel=</b>'
    ) AS excerpt
  FROM posts
  WHERE fts @@ plainto_tsquery('english', ${term})
`
```

### Code: Prisma's built-in `fullTextSearch` (limited preview)

```prisma
// schema.prisma — enable the preview feature
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch"]
}
```

```js
// Prisma's built-in search — uses tsquery syntax
const posts = await prisma.post.findMany({
  where: {
    title: {
      search: 'prisma & tutorial'  // raw tsquery syntax
    }
  }
})
// Compiles to: WHERE to_tsvector(title) @@ to_tsquery('prisma & tutorial')
// Limitations:
// - No stored tsvector — recomputes on every query (slow at scale)
// - No GIN index used (unless carefully set up)
// - No ts_rank for relevance sorting
// - No ts_headline for excerpts
```

---

## What's Really Happening

**PostgreSQL FTS is a pipeline:**
1. **Document** — raw text (`title || ' ' || body`)
2. **`to_tsvector`** — tokenizes, normalizes (stems), and removes stop words → `tsvector` (e.g. `'prisma':1 'tutori':3`)
3. **`to_tsquery` / `plainto_tsquery`** — parses the search query → `tsquery` (e.g. `'prisma' & 'tutori'`)
4. **`@@` operator** — matches `tsvector` against `tsquery` → `true`/`false`
5. **`ts_rank`** — scores how well a document matches the query

**Why stored `tsvector` + GIN index?** — `WHERE to_tsvector('english', title || body) @@ query` recomputes the tsvector for every row on every query — it's a full table scan. A `GENERATED ALWAYS AS ... STORED` column computes the tsvector at insert/update time and stores it. A GIN index on that column lets Postgres jump directly to matching rows without scanning everything.

**Prisma's `fullTextSearch` preview limitations:**
- Uses `to_tsvector` on the fly — cannot use a stored tsvector column
- Does not support `ts_rank` ordering
- Does not support `ts_headline`
- The search argument uses raw `tsquery` syntax (`&`, `|`, `!`, `<->`) which is unfriendly for end users. Prefer `websearch_to_tsquery` in raw SQL for user-facing search.

**Language configuration** — `to_tsvector('english', text)` uses the `english` text search configuration, which includes English-specific stemming (e.g., "running" and "run" are the same token). Choose the right language config for your content. Supabase supports all Postgres text search configurations.

> **Leaky abstraction alert:** Prisma's `fullTextSearch` preview generates `to_tsvector(field)` without the language config argument — it uses the database default. If your Supabase project's default config isn't `english`, you may get incorrect stemming. For production, always use raw SQL where you control the language configuration explicitly.

---

## Practice

1. Add a `fts tsvector` generated column to `posts` using a migration SQL file. Create a GIN index. Test with `EXPLAIN ANALYZE` to confirm the index is used.

2. Write a `searchPosts(query: string)` function using `$queryRaw` that: (a) uses `websearch_to_tsquery`, (b) returns results sorted by `ts_rank`, (c) returns a `ts_headline` excerpt for each result.

3. Compare Prisma's built-in `fullTextSearch` with the raw SQL approach. Run both against a posts table with 10,000 rows. Check the `EXPLAIN` output. Which uses an index?

---

### Documentation Links

- PostgreSQL full-text search: [Full Text Search](https://www.postgresql.org/docs/current/textsearch.html)
- PostgreSQL `tsvector` / `tsquery`: [Text Search Types](https://www.postgresql.org/docs/current/datatype-textsearch.html)
- PostgreSQL `ts_rank`: [Text Search Functions](https://www.postgresql.org/docs/current/textsearch-controls.html#TEXTSEARCH-RANKING)
- PostgreSQL `ts_headline`: [Highlighting Results](https://www.postgresql.org/docs/current/textsearch-controls.html#TEXTSEARCH-HEADLINE)
- PostgreSQL GIN indexes for FTS: [GIN Indexes](https://www.postgresql.org/docs/current/gin.html)
- Prisma `fullTextSearch` (preview): (verify: https://www.prisma.io/docs/orm/prisma-client/queries/full-text-search)
- Supabase full-text search guide: (verify: https://supabase.com/docs/guides/database/full-text-search)
