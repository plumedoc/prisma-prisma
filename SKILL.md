---
name: "prisma-prisma"
description: "Use when building, querying, migrating, or debugging a Node.js/TypeScript application that uses Prisma ORM (the `prisma` CLI, `@prisma/client`, `prisma.config.ts`, Prisma schema/`schema.prisma`, `@prisma/adapter-*` driver adapters, or Prisma Migrate/Studio). Covers writing a Prisma schema, configuring `prisma.config.ts`, generating and using Prisma Client (CRUD, transactions, raw/typed SQL, extensions), running migrations safely, and picking/using the right driver adapter."
---

# Working with Prisma ORM

Prisma ORM = a Prisma schema (data model + config) → generated, type-safe Prisma Client, plus a CLI (`prisma`) for migrations, introspection, and tooling. There is no runtime schema reflection: everything Prisma Client exposes is code generated ahead of time from `schema.prisma`, so **the client is only as current as the last `prisma generate` run.**

## The four files that matter in every project

1. **`schema.prisma`** (or a `prisma/schema/` folder of `*.prisma` files) — `datasource` (db provider), `generator client { provider = "prisma-client"; output = "..." }`, and `model`/`enum` blocks.
2. **`prisma.config.ts`** — the connection string and CLI behavior (migrations path, seed command, etc.), built with `defineConfig`/`env` from `prisma/config`. **Not** auto-populated from `.env`; load env vars yourself (`import 'dotenv/config'`, `node --env-file=.env`, Bun).
3. The **generated client** at the `output` path from the `generator client` block (e.g. `./generated/client`). Import `PrismaClient` from there, not from `@prisma/client`, when a custom `output` is set.
4. **`prisma/migrations/`** — SQL migration history managed by `prisma migrate`.

## Task: scaffold a new project

```bash
npm install prisma --save-dev && npm install @prisma/client
npx prisma init            # or: npx prisma init --datasource-provider mysql --with-model
```
`prisma init` writes a starter `schema.prisma`, a `prisma.config.ts`, `.env`, and `.gitignore`. Default flow targets a local Prisma Postgres (`prisma dev` spins up a zero-config local Postgres server); pass `--datasource-provider` for another database, `--url` for a specific connection string.

## Task: define/update the data model

Edit `schema.prisma` models directly, or get one automatically:
- From an existing database: `npx prisma db pull` (introspection; add `--force` to overwrite instead of merge, `--print` to preview without writing).
- From nothing: hand-write `model`/`enum` blocks, then apply with Migrate or `db push`.

Always run `npx prisma format` after hand-editing (normalizes formatting) and `npx prisma validate` if unsure the schema still parses.

**After every schema change, re-run `npx prisma generate`.** Forgetting this is the single most common source of "the types don't match the database" bugs — the generated client is a snapshot, not live.

## Task: apply schema changes to a database

- **Local development, want history**: `npx prisma migrate dev --name <description>`. Diffs schema vs. migration history, writes+applies a new migration folder under `prisma/migrations/`, then runs `generate`. Use `--create-only` to draft a migration without applying it (e.g. to hand-edit raw SQL first).
- **CI / staging / production**: `npx prisma migrate deploy`. Applies committed, pending migrations only — never creates new ones, never prompts.
- **Fast prototyping, no history wanted**: `npx prisma db push` (add `--accept-data-loss` to allow lossy changes non-interactively; `--force-reset` to drop and recreate first).
- **Fix broken migration state**: `npx prisma migrate status` to see what's applied/pending/failed, then `npx prisma migrate resolve --applied <name>` or `--rolled-back <name>` to reconcile history without re-running SQL.
- **Generate a migration/diff without applying anything** (e.g. to hand off a hotfix script): `npx prisma migrate diff --from-config-datasource --to-schema=schema.prisma --script`.

### Destructive-command safety (important for agents)

`prisma migrate reset`, `prisma db push --force-reset`, and (legacy) `db drop` **delete all data**. Never run these against anything but a throwaway/dev database, and never pass `--force`/`-f` to skip the confirmation prompt unless a human has explicitly asked for a full reset. Prisma itself gates these behind extra confirmation when it detects an AI agent is driving the CLI — respect that checkpoint rather than working around it.

## Task: query with Prisma Client

```ts
import { PrismaClient } from './generated/client'   // path = generator's `output`
import { PrismaPg } from '@prisma/adapter-pg'        // pick the adapter matching your DB/driver

const prisma = new PrismaClient({
  adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL }),
})

await prisma.user.findMany({ where: { email: { contains: '@acme.com' } }, include: { posts: true } })
await prisma.user.create({ data: { name: 'Alice', posts: { create: { title: 'Hi' } } } })
await prisma.post.update({ where: { id: 42 }, data: { published: true } })
await prisma.user.delete({ where: { id: 1 } })
```

Every model gets a camelCased delegate (`Post` → `prisma.post`) with `findUnique`, `findUniqueOrThrow`, `findFirst`, `findMany`, `create`, `createMany`, `update`, `updateMany`, `upsert`, `delete`, `deleteMany`, `count`, `aggregate`, `groupBy`.

**Driver adapters are the default/recommended construction path** in this codebase's own examples — always pass `adapter` explicitly rather than relying on a bundled native engine, and pick the adapter package matching the actual driver in use (`@prisma/adapter-pg` for `pg`, `@prisma/adapter-neon` for Neon, `@prisma/adapter-planetscale`, `@prisma/adapter-libsql`, `@prisma/adapter-better-sqlite3`, `@prisma/adapter-mariadb`, `@prisma/adapter-mssql`, `@prisma/adapter-d1` for Cloudflare D1/Workers). Cloudflare D1 has separate `index-node`/`index-workerd` builds — use the workerd one when deploying to Workers.

### Transactions

```ts
// Batch (all-or-nothing, independent operations):
const [a, b] = await prisma.$transaction([
  prisma.user.create({ data: { name: 'A' } }),
  prisma.user.create({ data: { name: 'B' } }),
])

// Interactive (sequential, dependent logic):
await prisma.$transaction(async (tx) => {
  await tx.account.update({ where: { id: 1 }, data: { balance: { decrement: 10 } } })
  await tx.account.update({ where: { id: 2 }, data: { balance: { increment: 10 } } })
}, { timeout: 10000 })
```
Use batch `$transaction` for independent writes; use the interactive callback form when a later query depends on an earlier one's result, and keep the callback short (it holds a DB connection/transaction open for its whole duration).

### Raw and typed SQL

```ts
const rows = await prisma.$queryRaw`SELECT * FROM "User" WHERE id = ${id}` // parameterized, safe
const n = await prisma.$executeRaw`UPDATE "User" SET cool = true WHERE id = ${id}`
```
Avoid `$queryRawUnsafe`/`$executeRawUnsafe` with any string built from user input — they take a plain string and are not automatically parameterized. Prefer the tagged-template `$queryRaw`/`$executeRaw` forms, or `$queryRawTyped(query)` with a query authored under the `typedSql`/`sql` directory and imported from the client's `sql` submodule, for compile-time-checked raw SQL.

### Errors worth branching on

`PrismaClientKnownRequestError` has a `.code` (e.g. `P2002` = unique constraint violation, `P2025` = record not found). Wrap risky writes:
```ts
try {
  await prisma.user.create({ data: { email } })
} catch (e) {
  if (e instanceof Prisma.PrismaClientKnownRequestError && e.code === 'P2002') {
    // handle duplicate
  }
  throw e
}
```

## Task: extend the client (shared query logic, computed fields, middleware-like hooks)

Use `prisma.$extends({ ... })` (client extensions) rather than the removed `$use` middleware API. Import extension-authoring helpers from `@prisma/client/extension` when building a reusable extension package.

## Pitfalls specific to Prisma

- **Stale client after schema edits**: always re-run `prisma generate` after touching `schema.prisma`; a stale generated client silently accepts/returns the old shape.
- **`.env` is not auto-loaded** by `prisma.config.ts` or by Prisma Client — an app that "works with the CLI" but fails at runtime (or vice versa) is often an env-loading mismatch between the two.
- **Connection string lives in `prisma.config.ts`, not (only) in the schema** — don't assume editing `datasource { url = ... }` in `schema.prisma` is sufficient in a project that has a `prisma.config.ts`.
- **Multiple `generator` blocks**: `prisma generate` runs all of them unless `--generator <name>` is passed; check which `output` you're actually importing from.
- **`migrate dev` is dev-only**: it can prompt for destructive resets when history and schema diverge; never run it unattended against production — use `migrate deploy` there.
- **Empty logical filters and `undefined`**: passing `undefined` for a filter value is treated as "omit this condition" (not "match null"); passing `where: { OR: [] }` has defined-but-easy-to-misread semantics — check current docs/behavior before relying on either in generated query code rather than assuming ORM-typical defaults.
- **Driver-adapter runtime mismatches**: an adapter built for Node (`index-node`) will not work in an edge/Workers runtime and vice versa (see D1, libsql `-web`/`-node` split) — match the adapter entry point to the deployment target.

## Quick CLI cheat sheet

```bash
npx prisma init                       # scaffold project
npx prisma format && npx prisma validate   # normalize + sanity-check schema
npx prisma generate                   # (re)generate client — run after every schema change
npx prisma migrate dev --name <msg>   # dev: create + apply + regenerate
npx prisma migrate deploy             # CI/prod: apply pending migrations only
npx prisma migrate status             # inspect migration state
npx prisma db pull                    # introspect existing DB into schema
npx prisma db push                    # prototype: sync schema to DB, no history
npx prisma studio                     # GUI data browser
npx prisma db seed                    # run configured seed command
```

For exhaustive flag lists and the full Prisma Client API surface, see the reference pages (`reference/cli.md`, `reference/client-api.md`, `reference/schema.md`, `reference/config.md`, `reference/driver-adapters.md`) shipped alongside this skill.

## Staying current

This skill was generated by plumedoc from `prisma/prisma@cda80a4488b7b551c36bf09ca2e8303ef9509da4` on 2026-07-14.
It carries task-level knowledge only. For exhaustive reference, anything outside
the ground it covers, or if `prisma/prisma` has moved past this commit,
consult the live plumedoc docs at https://plumedoc.com/prisma/prisma (and its `llms.txt`), which track the source.
