# Prisma Setup for Existing Projects

[![Node.js](https://img.shields.io/badge/Node.js-v18+-green.svg)](https://nodejs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue.svg)](https://www.typescriptlang.org/)
[![Prisma](https://img.shields.io/badge/Prisma-5.x-blueviolet.svg)](https://www.prisma.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-blue.svg)](https://www.postgresql.org/)

A concise guide to integrate **Prisma** into an existing **Express** or similar Node.js project with **PostgreSQL**, **TypeScript**, **migrations**, and optional seeding. This setup ensures a robust database workflow, including safe migrations and Prisma Studio for development.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Install Prisma & Client](#install-prisma--client)
- [Initialize Prisma](#initialize-prisma)
- [Configure `.env`](#configure-env)
- [Define Your Data Model](#define-your-data-model)
- [Migrate Your Database](#migrate-your-database)
- [Generate Prisma Client](#generate-prisma-client)
- [Use Prisma Client in Code](#use-prisma-client-in-code)
- [Optional: Seeding](#optional-seeding)
- [Prisma Studio](#prisma-studio)
- [Troubleshooting & Tips](#troubleshooting--tips)
- [Commands Cheat Sheet](#commands-cheat-sheet)
- [Additional Resources](#additional-resources)

---

<details>
<summary><h2 id="overview">Overview</h2></summary>

This guide integrates **Prisma** into an existing Node.js project, enabling type-safe database operations with **PostgreSQL**. Key features include:

- **Prisma ORM** for database queries
- **TypeScript** for type safety
- **Prisma Migrations** for schema evolution
- **Prisma Studio** for interactive database management
- **Optional seeding** for test data
- **Safe migration workflows** to avoid data loss

The setup assumes an existing project (e.g., Express) and uses PostgreSQL, but it’s adaptable to other databases supported by Prisma.

</details>

---

<details>
<summary><h2 id="prerequisites">Prerequisites</h2></summary>

Ensure you have:

- **Node.js** (v18+ recommended) or **Bun**
- **PostgreSQL** (v15+, running locally or via Docker, accessible at `localhost:5432`)
- **TypeScript** (optional, but recommended for type safety)
- **Git** (for version control)
- **Existing Node.js project** (e.g., Express with `package.json`)

For PostgreSQL, create a database named `next_blog`:
```bash
psql -U postgres -c "CREATE DATABASE next_blog;"
```

Alternatively, use Docker (see [Additional Resources](#additional-resources)).

</details>

---

<details>
<summary><h2 id="install-prisma--client">Install Prisma & Client</h2></summary>

Inside your project folder, install Prisma and Prisma Client:

```bash
bun add prisma @prisma/client --dev
# or with npm
npm install prisma @prisma/client --save-dev
```

> **Note**: Installing as dev dependencies is typical since `@prisma/client` is generated during development.

</details>

---

<details>
<summary><h2 id="initialize-prisma">Initialize Prisma</h2></summary>

Run:

```bash
npx prisma init
```

This creates:
```
📂 prisma/
   └── schema.prisma
.env
```

- `schema.prisma`: Defines your database schema and configuration.
- `.env`: Stores environment variables (e.g., database URL).

Add `.env` to `.gitignore`:
```
.env
```

</details>

---

<details>
<summary><h2 id="configure-env">Configure `.env`</h2></summary>

Edit `.env` with your PostgreSQL connection string:

```env
DATABASE_URL="postgresql://postgres:your_password@localhost:5432/next_blog?schema=public"
```

> **Note**: Ensure the database `next_blog` exists and PostgreSQL is running. Replace `your_password` with your actual PostgreSQL password.

</details>

---

<details>
<summary><h2 id="define-your-data-model">Define Your Data Model</h2></summary>

Edit `prisma/schema.prisma` to define your models. Example for a blog project:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id       Int      @id @default(autoincrement())
  email    String   @unique
  name     String?
  posts    Post[]
  role     Role     @default(USER)
  status   UserStatus @default(ACTIVE)
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
}

enum Role {
  USER
  ADMIN
}

enum UserStatus {
  ACTIVE
  INACTIVE
}
```

</details>

---

<details>
<summary><h2 id="migrate-your-database">Migrate Your Database</h2></summary>

Apply the schema to your database:

```bash
npx prisma migrate dev --name init
```

This:
- Creates `prisma/migrations/YYYYMMDDHHmmss_init/migration.sql`
- Applies the migration to your database
- Generates the Prisma Client

If Prisma detects **schema drift** (mismatch between database and migrations), you may need to reset:

```bash
npx prisma migrate reset
```

> **Warning**: `migrate reset` drops all data in the development database. Use cautiously.

For safer migrations, see [Troubleshooting & Tips](#troubleshooting--tips).

</details>

---

<details>
<summary><h2 id="generate-prisma-client">Generate Prisma Client</h2></summary>

The Prisma Client is auto-generated after migrations. Regenerate it manually if you update `schema.prisma`:

```bash
npx prisma generate
```

This updates `@prisma/client` for use in your code.

</details>

---

<details>
<summary><h2 id="use-prisma-client-in-code">Use Prisma Client in Code</h2></summary>

### `src/db.ts`
Initialize Prisma Client:

```ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();
export default prisma;
```

### Example Usage
Integrate Prisma into your existing project (e.g., Express routes):

```ts
import prisma from './db';

// Create a user
const user = await prisma.user.create({
  data: { email: 'test@example.com', name: 'Sarwar' },
});

// Fetch all users
const users = await prisma.user.findMany();
console.log(users);
```

Example Express route (`src/index.ts`):

```ts
import express, { Request, Response } from 'express';
import prisma from './db';

const app = express();
app.use(express.json());

app.post('/users', async (req: Request, res: Response) => {
  const { email, name } = req.body;
  try {
    const user = await prisma.user.create({ data: { email, name } });
    res.status(201).json(user);
  } catch (err) {
    res.status(400).json({ error: 'Invalid or duplicate data' });
  }
});

app.listen(4000, () => console.log('Server running at http://localhost:4000'));
```

</details>

---

<details>
<summary><h2 id="optional-seeding">Optional: Seeding</h2></summary>

Create `prisma/seed.ts` to populate test data:

```ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  await prisma.user.create({
    data: {
      email: 'admin@example.com',
      name: 'Admin User',
      role: 'ADMIN',
      status: 'ACTIVE',
    },
  });
  await prisma.user.create({
    data: {
      email: 'user@example.com',
      name: 'User',
      role: 'USER',
      status: 'ACTIVE',
    },
  });
}

main()
  .then(() => console.log('Seed data created'))
  .catch((e) => console.error(e))
  .finally(async () => {
    await prisma.$disconnect();
  });
```

Configure seeding in `package.json`:

```json
"prisma": {
  "seed": "ts-node prisma/seed.ts"
}
```

Run the seed script:

```bash
npx prisma db seed
```

> **Note**: Seeding runs automatically with `npx prisma migrate reset`.

</details>

---

<details>
<summary><h2 id="prisma-studio">Prisma Studio</h2></summary>

Launch Prisma Studio to browse/edit your database:

```bash
npx prisma studio
```

Access it at `http://localhost:5555`.

Add to `package.json` scripts for convenience:

```json
"scripts": {
  "studio": "npx prisma studio"
}
```

</details>

---

<details>
<summary><h2 id="troubleshooting--tips">Troubleshooting & Tips</h2></summary>

- **Error: P1000 Authentication Failed**
  - Verify `DATABASE_URL` credentials in `.env`.
  - Ensure PostgreSQL is running and accepts connections.
  - Use `postgresql://` protocol.

- **bash: prisma: command not found**
  - Use `npx prisma` or `bunx prisma`, or install globally: `npm i -g prisma`.

- **Schema Drift Detected**
  - If `migrate dev` fails due to drift:
    1. Create a migration without applying:
       ```bash
       npx prisma migrate dev --create-only --name fix-drift
       ```
    2. Edit `prisma/migrations/YYYYMMDDHHmmss_fix-drift/migration.sql` to match the database.
    3. Apply:
       ```bash
       npx prisma migrate dev
       ```
    4. Alternatively, mark as applied:
       ```bash
       npx prisma migrate resolve --applied YYYYMMDDHHmmss_fix-drift
       ```

- **Prototyping Tip**: Use `npx prisma db push` for rapid schema changes without migration files (dev only, not for production).

- **Performance Tip**: Reuse the Prisma Client instance (`src/db.ts`) to avoid connection overhead.

</details>

---

<details>
<summary><h2 id="commands-cheat-sheet">Commands Cheat Sheet</h2></summary>

| Command                              | Description                                      |
|--------------------------------------|--------------------------------------------------|
| `npx prisma init`                    | Initialize Prisma (`schema.prisma` + `.env`)     |
| `npx prisma migrate dev --name X`    | Create and apply migration (dev)                |
| `npx prisma migrate dev --create-only --name X` | Create migration without applying      |
| `npx prisma migrate deploy`          | Apply migrations (production)                   |
| `npx prisma db push`                 | Sync schema to DB (no migration files, dev only) |
| `npx prisma db seed`                 | Run seed script                                 |
| `npx prisma studio`                  | Launch Prisma Studio (`localhost:5555`)         |
| `npx prisma generate`                | Generate Prisma Client                           |

</details>

---

<details>
<summary><h2 id="additional-resources">Additional Resources</h2></summary>

### Full `package.json` Example
```json
{
  "name": "your-project",
  "version": "1.0.0",
  "scripts": {
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "studio": "npx prisma studio",
    "migrate:dev": "npx prisma migrate dev",
    "migrate:create": "npx prisma migrate dev --create-only",
    "migrate:deploy": "npx prisma migrate deploy",
    "migrate:reset": "npx prisma migrate reset",
    "db:push": "npx prisma db push",
    "db:seed": "npx prisma db seed",
    "prisma:generate": "npx prisma generate"
  },
  "dependencies": {
    "@prisma/client": "^5.0.0",
    "express": "^4.18.2"
  },
  "devDependencies": {
    "@types/express": "^4.17.17",
    "@types/node": "^18.15.0",
    "nodemon": "^3.0.1",
    "prisma": "^5.0.0",
    "ts-node": "^10.9.1",
    "typescript": "^5.0.4"
  },
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

### TypeScript Configuration
Create `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*", "prisma/seed.ts"],
  "exclude": ["node_modules"]
}
```

### Docker Setup
Run PostgreSQL with Docker:
```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: your_password
      POSTGRES_DB: next_blog
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Run:
```bash
docker-compose up -d
```

### Generate Files
If you need files like `index.ts`, `db.ts`, `seed.ts`, `schema.prisma`, `tsconfig.json`, or `docker-compose.yml`, let me know, and I can provide them as snippets or a zip.

</details>

---

✅ Your project is now **Prisma-ready**:
1. Define schema in `prisma/schema.prisma`
2. Run `npx prisma migrate dev`
3. Use `prisma` client in your app
