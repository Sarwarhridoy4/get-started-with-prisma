

# Express + Prisma + Prisma Studio — Fresh Start

[![Node.js](https://img.shields.io/badge/Node.js-v18+-green.svg)](https://nodejs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue.svg)](https://www.typescriptlang.org/)
[![Prisma](https://img.shields.io/badge/Prisma-5.x-blueviolet.svg)](https://www.prisma.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-blue.svg)](https://www.postgresql.org/)

A concise, step-by-step guide to bootstrap a modern **Express** application with **Prisma**, **migrations**, **Prisma Studio**, and optional seeding. Tailored for local development with **PostgreSQL** and **TypeScript**, this setup includes a robust migration workflow (including "remigrate" without full DB reset) and a production-ready configuration.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Quickstart](#quickstart)
4. [Project Layout](#project-layout)
5. [Installation & Initialization](#installation--initialization)
6. [Configure `.env` and `schema.prisma`](#configure-env-and-schemaprisma)
7. [Initial Migration & Prisma Client](#initial-migration--prisma-client)
8. [Remigrate Workflow (Safe Schema Updates)](#remigrate-workflow-safe-schema-updates)
9. [Seeding (Optional)](#seeding-optional)
10. [Example Express Server (TypeScript)](#example-express-server-typescript)
11. [Useful `package.json` Scripts](#useful-packagejson-scripts)
12. [Prisma Studio](#prisma-studio)
13. [Production Notes](#production-notes)
14. [Troubleshooting & Tips](#troubleshooting--tips)
15. [Commands Cheat Sheet](#commands-cheat-sheet)
16. [Additional Resources](#additional-resources)

---

## Overview

This guide sets up a minimal, scalable Express server integrated with Prisma for database access. Key features include:

- **TypeScript** for type-safe code
- **Prisma ORM** for database operations
- **Prisma Migrations** with a safe "remigrate" workflow to evolve schemas without data loss
- **Prisma Studio** for interactive database browsing/editing
- **Optional seeding** for initial data
- **PostgreSQL** as the default database (adaptable to MySQL, SQLite, etc.)

The setup is optimized for **local development** but includes production considerations.

---

## Prerequisites

Before starting, ensure you have:

- **Node.js** (v18+ recommended) or **Bun**
- **PostgreSQL** (v15+, running locally or via Docker, accessible at `localhost:5432`)
- **Git** (for version control)
- **Optional (TypeScript)**: `tsc`, `ts-node`, `nodemon` for development

For PostgreSQL setup, you can use a local installation or a Docker container (see [Additional Resources](#additional-resources) for a `docker-compose.yml` example).

---

## Quickstart

Get up and running with these commands. Choose your package manager:

### Using npm
```bash
mkdir express-prisma-app && cd express-prisma-app
npm init -y
npm install express @prisma/client
npm install -D prisma typescript ts-node nodemon @types/express @types/node
npx prisma init
# Edit .env and prisma/schema.prisma (see below)
npx prisma migrate dev --name init
npx prisma generate
npm run dev
# In a new terminal
npx prisma studio
```

### Using Bun
```bash
mkdir express-prisma-app && cd express-prisma-app
bun init -y
bun add express @prisma/client
bun add -d prisma typescript ts-node nodemon @types/express @types/node
bunx prisma init
# Edit .env and prisma/schema.prisma (see below)
bunx prisma migrate dev --name init
bunx prisma generate
bun run dev
# In a new terminal
bunx prisma studio
```

---

## Project Layout

Recommended structure for clarity and scalability:

```
express-prisma-app/
├── prisma/
│   ├── schema.prisma           # Prisma schema definition
│   ├── migrations/             # Auto-generated migration files
│   └── seed.ts                # Optional seeding script
├── src/
│   ├── index.ts               # Express server entry point
│   └── db.ts                  # Prisma Client initialization
├── .env                       # Environment variables
├── .gitignore                 # Git ignore file
├── package.json               # Dependencies and scripts
├── tsconfig.json              # TypeScript configuration
└── README.md                  # This file
```

---

## Installation & Initialization

1. **Create project directory**:
   ```bash
   mkdir express-prisma-app && cd express-prisma-app
   ```

2. **Initialize project**:
   - For npm: `npm init -y`
   - For Bun: `bun init -y`

3. **Install dependencies**:
   - Core: `express`, `@prisma/client`
   - Dev: `prisma`, `typescript`, `ts-node`, `nodemon`, `@types/express`, `@types/node`
   - See [Quickstart](#quickstart) for exact commands.

4. **Initialize Prisma**:
   ```bash
   npx prisma init
   ```
   This creates `prisma/schema.prisma` and `.env`.

5. **Set up TypeScript**:
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

6. **Update `.gitignore`**:
   Ensure `.env` and build artifacts are ignored:
   ```
   node_modules/
   dist/
   .env
   ```

---

## Configure `.env` and `schema.prisma`

### `.env`
Configure your database connection in `.env`:
```env
DATABASE_URL="postgresql://postgres:your_password@localhost:5432/express_prisma?schema=public"
```

> **Note**: Ensure the database (`express_prisma`) exists in PostgreSQL. Create it with:
> ```bash
> psql -U postgres -c "CREATE DATABASE express_prisma;"
> ```

### `schema.prisma`
Define your data model. Example with `User` and `Post` models:
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role {
  USER
  ADMIN
}

enum UserStatus {
  ACTIVE
  INACTIVE
}

model User {
  id       Int        @id @default(autoincrement())
  email    String     @unique
  name     String?
  role     Role       @default(USER)
  status   UserStatus @default(ACTIVE)
  posts    Post[]
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
}
```

---

## Initial Migration & Prisma Client

1. **Create and apply initial migration**:
   ```bash
   npx prisma migrate dev --name init
   ```
   This generates `prisma/migrations/YYYYMMDDHHmmss_init/migration.sql`, applies it to the database, and creates the Prisma Client.

2. **Generate Prisma Client** (if schema changes later):
   ```bash
   npx prisma generate
   ```

---

## Remigrate Workflow (Safe Schema Updates)

When updating `schema.prisma`, use these workflows to apply changes safely, preserving data where possible.

### A. Standard Development Workflow
For linear development with a healthy migration history:
1. Edit `schema.prisma` (e.g., add a new field or model).
2. Run:
   ```bash
   npx prisma migrate dev --name update-model
   ```
   This generates and applies a new migration.

### B. Customize Migration SQL
To avoid data loss from destructive changes:
1. Generate migration without applying:
   ```bash
   npx prisma migrate dev --create-only --name update-model
   ```
2. Edit the SQL in `prisma/migrations/YYYYMMDDHHmmss_update-model/migration.sql` to preserve data (e.g., use `ALTER TABLE` or temporary tables).
3. Apply the migration:
   ```bash
   npx prisma migrate dev
   ```

### C. Prototyping (No Migration History)
For rapid schema changes during early development:
```bash
npx prisma db push
```
> **Warning**: `db push` syncs the schema directly without migration files. Use only for prototyping, not production.

### D. Handling Schema Drift
If Prisma detects a mismatch between your database and migration history:
- **Option 1: Reset DB** (dev only, wipes data):
  ```bash
  npx prisma migrate reset
  ```
- **Option 2: Manual Resolution**:
  1. Generate a migration to align with the current DB:
     ```bash
     npx prisma migrate dev --create-only --name fix-drift
     ```
  2. Edit the SQL to match the database state.
  3. Apply or mark as applied:
     ```bash
     npx prisma migrate resolve --applied YYYYMMDDHHmmss_fix-drift
     ```

> **Best Practice**: Use `migrate dev` for most cases. Reserve `db push` for prototyping and `--create-only` for complex changes requiring custom SQL.

---

## Seeding (Optional)

Create a seed script to populate initial data.

### `prisma/seed.ts`
```ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  await prisma.user.create({
    data: {
      email: 'admin@example.com',
      name: 'Admin',
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
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

### Configure Seeding
In `package.json`, add:
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

---

## Example Express Server (TypeScript)

### `src/db.ts`
Initialize Prisma Client:
```ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();
export default prisma;
```

### `src/index.ts`
Basic Express server with CRUD endpoints:
```ts
import express, { Request, Response } from 'express';
import prisma from './db';

const app = express();
app.use(express.json());

// Create a user
app.post('/users', async (req: Request, res: Response) => {
  const { name, email, role, status } = req.body;
  try {
    const user = await prisma.user.create({
      data: { name, email, role, status },
    });
    res.status(201).json(user);
  } catch (err) {
    res.status(400).json({ error: 'Invalid or duplicate data' });
  }
});

// Get all users
app.get('/users', async (req: Request, res: Response) => {
  const users = await prisma.user.findMany();
  res.json(users);
});

// Get user by ID
app.get('/users/:id', async (req: Request, res: Response) => {
  const { id } = req.params;
  const user = await prisma.user.findUnique({ where: { id: parseInt(id) } });
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});

app.listen(4000, () => console.log('Server running at http://localhost:4000'));
```

---

## Useful `package.json` Scripts

Add these to `package.json` for convenience:
```json
{
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
  }
}
```

Example usage:
```bash
npm run migrate:create -- --name add-field
npm run dev
```

---

## Prisma Studio

Run Prisma Studio to browse and edit your database:
```bash
npx prisma studio
# or
npm run studio
```
Access it at `http://localhost:5555`.

---

## Production Notes

- **Use `migrate deploy`**: In production, apply migrations with:
  ```bash
  npx prisma migrate deploy
  ```
- **Environment Variables**: Ensure `DATABASE_URL` is set in your production environment (e.g., via CI/CD or hosting provider).
- **Prisma Client**: Generate the client before building (`npx prisma generate`).
- **Avoid `db push` and `migrate reset`**: These are for development only.
- **Connection Pooling**: For PostgreSQL, configure `?pgbouncer=true` in `DATABASE_URL` if using a connection pooler like PgBouncer.

---

## Troubleshooting & Tips

- **Error: P1000 Authentication Failed**
  - Verify `DATABASE_URL` credentials in `.env`.
  - Ensure PostgreSQL is running and accepts connections.
  - Use `postgresql://` protocol.

- **bash: prisma: command not found**
  - Use `npx prisma` or `bunx prisma`, or install globally: `npm i -g prisma`.

- **Schema Drift Detected**
  - See [Remigrate Workflow](#remigrate-workflow-safe-schema-updates) for resolution options.
  - Use `--create-only` to craft custom migrations or `prisma migrate resolve` to mark migrations as applied.

- **Performance Tip**: Limit Prisma Client instances by reusing `src/db.ts` across your app.

- **Debugging Migrations**: Check `prisma/migrations/*` for SQL files and logs in Prisma commands.

---

## Commands Cheat Sheet

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
| `npm run dev`                        | Start development server                        |
| `npm run build && npm run start`     | Build and start production server               |

---

## Additional Resources

### Full `package.json` Example
```json
{
  "name": "express-prisma-app",
  "version": "1.0.0",
  "main": "dist/index.js",
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

### Docker Setup
To run PostgreSQL locally with Docker:
```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: your_password
      POSTGRES_DB: express_prisma
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