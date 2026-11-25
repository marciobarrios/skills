---
name: check-migrations
description: Find and check the database migrations in a branch or PR: whether something applies them automatically, what production has already applied, whether they're safe before and after merge, lock cost, security, and rollback. Gives a verdict (applied automatically, run before merge, run after merge, or not safe) and writes the PR's Migrations section. Use when the user asks whether a PR's migrations are safe, when to run them, or how to apply them, and when another skill (such as autopilot or safe-merge) needs that verdict.
---

# check-migrations

Migrations are easy to miss in a PR, and the user usually applies them by
hand, **before** merging. Never apply a migration to production or any
shared database yourself. Your job is to find them, check them, and make
sure the user can't miss them.

Project config may set the migration paths, the command the user runs to
apply them, and a read-only way to see which ones production has applied:
`AUTOPILOT.md` at the repo root, or an `## Autopilot` section in
`AGENTS.md` / `CLAUDE.md`. It wins over the defaults below.

The target is a PR (number or URL) or the current branch. Compare it with
its base (`origin/<base>`, after `git fetch origin`).

## 1. Detect

List every migration the branch adds or changes: `git diff --name-only
origin/<base>...HEAD` (or `gh pr diff <n> --name-only`) against the
config's migration paths, or the usual ones (`supabase/migrations/`,
`prisma/migrations/`, `drizzle/`, `migrations/`, `db/migrate/`,
`alembic/versions/`).

Also check whether the ORM schema changed (`schema.prisma`, Drizzle
schema files) without a matching migration. That's a bug: fix it when
the branch is yours to change, otherwise report it as a blocker.

No migrations: say so and stop.

## 2. Automatic apply?

Find out whether something already applies migrations on merge or
deploy: project config first, then `.github/workflows/` (and other CI
config) for steps like `supabase db push`, `supabase migration up`,
`prisma migrate deploy`, `drizzle-kit migrate`, `knex migrate:latest`,
`rails db:migrate`, `alembic upgrade`; the build or deploy scripts in
`package.json`, `vercel.json` / `vercel.ts` (e.g. `prisma migrate deploy
&& next build`); and platform integrations (Supabase GitHub integration,
Neon/Vercel).

Note *when* it runs (before the new code deploys, in parallel with it, or
after) and whether it needs a manual approval (e.g. a protected GitHub
environment). If it's automatic with no approval, that setup fixes the
order: step 4 checks the order is safe.

## 3. Production's state, read-only

When there's a way (project config, Supabase MCP `list_migrations` /
`list_tables`, Neon MCP `describe_table_schema`), check:

- which migrations production has already applied,
- whether older migrations from *other* PRs are still pending (call those
  out: they must go first),
- the size of the tables touched.

Only read. If there's no way to check, say so.

## 4. Safety in both orders

Production keeps running the old code until the merge deploys, so check:

- **Before merge:** does the **old** code still work against the **new**
  schema? Additive changes are usually safe: new tables, nullable columns
  or columns with a default, new indexes, functions, views. Drops,
  renames, type changes, new `NOT NULL` without a default, and tighter
  constraints or RLS that the old code violates are not.
- **After merge:** does the **new** code work against the **old** schema
  during the deploy window?
- **Cost:** locks and rewrites on large tables (an index without
  `CONCURRENTLY`, a column type change, backfills), duration, statements
  that can't run in a transaction.
- **Security:** RLS, grants, and policies that widen access.
- **Rollback:** is there a down migration or a written manual rollback,
  and does it lose data?

## 5. Verdict

One of:

- **Applied automatically:** step 2 found an automatic apply and its
  order is safe per step 4. If that order isn't safe (e.g. the workflow
  migrates after the deploy, but the new code needs the new schema),
  treat it as "not safe in either order".
- **Run before merge:** the default when it's safe.
- **Run after merge:** the new code tolerates the old schema, but the old
  code would break on the new one.
- **Not safe in either order:** propose backward-compatible steps (expand
  now, contract in a follow-up). Make that change only when the branch is
  yours to change and it's small; otherwise ask the user.

## 6. Write it down

Under a **Migrations** heading (in the PR body when there's a PR; keep
everything else in the body, including any `<!-- autopilot:state … -->`
marker, unchanged):

- each file, and what it does in one line,
- the verdict and why,
- lock and duration risk,
- rollback,
- pending migrations from other PRs,
- the exact command to apply it, or the workflow or script that applies
  it automatically.

Then tell the user the same, putting it first in your message: the files,
the verdict, and the exact command. Never let migrations get buried.

## 7. After the user applies it

When the user says it's applied, verify it read-only with step 3's method
when you can, and say whether you could. After a merge with an automatic
apply, confirm the apply ran and succeeded (the workflow run or deploy
logs, plus step 3's check). A failed or missing apply is a regression:
tell the user right away. Never revert a migration automatically.
