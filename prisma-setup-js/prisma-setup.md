# Prisma ORM (JavaScript Version)

This setup considers you already have a node.js project initiated.

Run the commands below step by step.

## 1. Install dependencies

```js
npm install prisma @types/pg --save-dev
npm install @prisma/client @prisma/adapter-pg pg dotenv
```

## 2. Update `package.json` to enable ESM

```js
{
  "type": "module"
}
```

## 3. Initialize Prisma

```js
npx prisma init --datasource-provider postgresql --generator-provider prisma-client-js --output ../generated/prisma
```

## 4. Rename `prisma.config.ts` to `prisma.config.js`

```
prisma.config.ts -> prisma.config.js
```
### 4.1. And do the following in the same file (`prisma.config.js`):
Replace 
```js
datasource: {
    url: process.env["DATABASE_URL"],
  }
```
with
```js
datasource: {
    url: process.env["DIRECT_URL"],
  }
```

## 5. Update your `.env` file with your PostgreSQL connection string

```js
// .env
# Connect to Postgres via the shared transaction-mode pooler (IPv4-only)
DATABASE_URL="postgresql://postgres.username:[YOUR-PASSWORD]@supabase.com:6543/postgres?pgbouncer=true"

# Connect to Postgres via the shared session-mode pooler (used for migrations)
DIRECT_URL="postgresql://postgres.username:[YOUR-PASSWORD]@supabase.com:5432/postgres"
```

## 6. Add models to `prisma/schema.prisma`

```js
model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id        Int     @id @default(autoincrement())
  title     String
  content   String?
  published Boolean @default(false)
  author    User    @relation(fields: [authorId], references: [id])
  authorId  Int
}
```

## 7. Run the migration

```js
npx prisma migrate dev --name init
```

## 8. Run the generation

```js
npx prisma generate
```

## 9. Instantiate Prisma Client in `lib/prisma.js`

```js
import "dotenv/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "../generated/prisma/index.js"; // the .js is mandatory since we are using ESM

const connectionString = `${process.env.DATABASE_URL}`;

const adapter = new PrismaPg({ connectionString });
const prisma = new PrismaClient({ adapter });

export { prisma };
```

## 10. Write your query in `query.js`

```js
import { prisma } from "./lib/prisma.js"; // the .js is mandatory since we are using ESM

async function main() {
  // Create a new user with a post
  const user = await prisma.user.create({
    data: {
      name: "Alice",
      email: "alice@prisma.io",
      posts: {
        create: {
          title: "Hello World",
          content: "This is my first post!",
          published: true,
        },
      },
    },
    include: {
      posts: true,
    },
  });
  console.log("Created user:", user);

  // Fetch all users with their posts
  const allUsers = await prisma.user.findMany({
    include: {
      posts: true,
    },
  });
  console.log("All users:", JSON.stringify(allUsers, null, 2));
}

main()
  .then(async () => {
    await prisma.$disconnect();
  })
  .catch(async (e) => {
    console.error(e);
    await prisma.$disconnect();
    process.exit(1);
  });
```

## 11. Run `node query.js`

References:

- https://www.prisma.io/docs/prisma-orm/quickstart/postgresql
- https://www.prisma.io/docs/cli/init#specify-a-generator-provider
