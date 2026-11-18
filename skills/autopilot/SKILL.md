---
name: autopilot
description: Use when the user's message starts with "autopilot" followed by a Linear issue URL or ID (e.g. "autopilot https://linear.app/acme/issue/ABC-123/..." or "autopilot ABC-123"). Implements the issue end to end, verifies it, gets an independent review, attaches evidence to a PR, assesses merge risk, auto-merges when it is safe, and monitors production afterwards.
---

# autopilot

Take one Linear issue from "Todo" to a merged, monitored PR without
hand-holding:

read → plan → capture *before* → implement → verify → capture *after* →
review (sol-loop) → PR → CI → Preview → merge-risk → merge or hand off →
monitor production.

Arguments after the issue are flags in plain words. Honor them:

- `no merge` / `draft` — open the PR, never merge.
- `merge medium` — allow auto-merge at medium risk too.
- Anything else is extra context for the implementation.

Assume the user is not watching this thread. Your updates and the final
message go unread until long after you stop, so stay active until the
report (step 11) and never end a turn with a plain-text question (see
"Asking the user").

## Testing permission

An `autopilot` request is the permission for the tests and evidence this
skill needs. If a project file or other instruction says to ask before a
test method, or to leave testing to the user, this request replaces that
rule for this run. That covers computer use, browser automation
(`agent-browser`), simulators and devices, running the app or a dev
server, screenshots, recordings, and visual checks. List the methods used
in the PR's **Verification** section.

Rules about *how* to test still apply: follow project rules that name the
correct command, fixture, account, or check. This permission only removes
the ask-first or leave-it-to-the-user step. It covers only the tests this
issue needs. It does not cover production writes, data deletion, or
anything under "Asking the user", and it does not bypass a harness or
system approval prompt.

## 0. Project config and resuming

Look for project overrides, in order: `AUTOPILOT.md` at the repo root, then
an `## Autopilot` section in `AGENTS.md` / `CLAUDE.md`. It may set: dev
server command and URL, test/lint/typecheck commands, E2E command, iOS
scheme and simulator, base branch, merge method, paths that must never
auto-merge, extra merge rules, Vercel Preview check depth (`full`,
`read-only`, or `skip`), production URLs and monitoring, and migrations
(migration paths, the command the user runs to apply them, and a
read-only way to see which ones production has applied). Project config
wins over this file. When absent, infer from `package.json` scripts,
`Makefile`, `*.xcodeproj`, CI workflow files, and the README.

**Resume, don't restart.** Runs get interrupted (turn ends while waiting
on CI, context gets summarized, a new session). Progress lives in two
places, kept current after every step:

- `.autopilot/state.md` in the worktree (excluded from git, see step 2):
  issue, branch, PR number, current step, fix attempts used, sol-loop
  ledger, evidence files, Preview URL, merge-risk result.
- A hidden marker in the PR body:
  `<!-- autopilot:state step=<step> fix-attempts=<n> review-rounds=<n> -->`.

When invoked for an issue that already has an open autopilot PR (or a
worktree on its branch, found with `git worktree list`), work in that
worktree, read both, verify them against reality (`gh pr view`, `git
log`), and continue from the recorded step.

## 1. Read and vet the issue

1. Parse the identifier (`ABC-123`) from the URL or argument.
2. Fetch it with the Linear MCP (`get_issue`). If one Linear server can't
   find it, try the other one. Also read `list_comments`, and pass any
   embedded images to `extract_images` — mockups and bug screenshots are
   requirements.
3. **Stop and ask the user** (see "Asking the user"; don't start work) when the issue is
   Done/Canceled, assigned to someone else and In Progress, or already
   has an open PR linked (unless it's this skill's PR: then resume).
4. Follow linked parent issues, sub-issues, documents, and Figma/URLs when
   they carry requirements.
5. Write down, for yourself: goal, acceptance criteria (explicit or
   derived), out of scope, open questions.
6. Ambiguity: if a reasonable engineer could pick a default, pick it and
   list it under **Assumptions** in the PR. If the issue is genuinely
   blocked (contradictory, missing a decision only a human can make),
   comment the question on the Linear issue, then ask the user (see
   "Asking the user").
7. **Size check.** If the work clearly spans several independent areas,
   or it's clearly beyond ~800 changed lines (excluding generated
   files, lockfiles, snapshots, and test fixtures; don't stop on a
   borderline estimate), don't build one huge PR: propose a split into
   sub-issues (title + one-line scope each) in a Linear comment, then ask
   the user (see "Asking the user") whether to create them and which slice
   to build first. If the size only becomes clear mid-implementation,
   stop at a coherent slice, open the PR with `Refs ABC-123` instead of
   `Fixes`, and propose the remaining sub-issues.
8. Move the issue to the team's "In Progress" status and assign it to `me`
   (check `list_issue_statuses` for the team's real names).

## 2. Worktree, branch, and evidence folder

**Always work in a dedicated git worktree**, never in the user's main
checkout: other autopilots and the user may be working in parallel. Don't
switch branches, stash, reset, or commit in the main checkout.

1. Branch name: the issue's `gitBranchName` from Linear (keeps the
   Linear ↔ GitHub link automatic); fall back to `<id-lowercase>-<slug>`.
2. `git fetch origin`, then get a worktree for that branch, created from
   the up-to-date `origin/<base>`, in this order:
   - **Already inside a linked worktree** (`git rev-parse --git-dir`
     differs from `--git-common-dir`; e.g. a T3 Code or Codex worktree
     thread): this is the worktree. Never create another one from inside
     it. Then:
     - On this issue's branch: a previous run, resume (step 0).
     - No commits of its own (`git log origin/<base>..HEAD` empty) and no
       uncommitted changes (`git status --porcelain`, ignoring
       `.autopilot/`): bring it up to date with `git merge --ff-only
       origin/<base>`, and rename its branch to the Linear name
       (`git branch -m <branch>`) if it still has the harness's
       placeholder name.
     - Anything else (commits or uncommitted changes unrelated to this
       issue): don't stash, reset, or discard them, and don't start a
       new worktree. Ask the user (see "Asking the user") whether to
       build on top of that work, or to stop so they can start a fresh
       thread.
   - In T3 Code with no worktree yet (`t3_worktree_status`): move this
     thread into one with `t3_worktree_handoff` (`branch` = the Linear
     name, `startFromOrigin: true`, `continuationPrompt: "autopilot
     <issue-id>: resume at step 2.3 (worktree setup)"`). It must be the
     last call of the turn; the work continues there, and T3 keeps the
     thread's diff and PR view bound to the worktree. Never create a
     worktree from the shell in T3: the thread wouldn't see it.
   - The harness has another worktree tool (Claude Code `EnterWorktree`):
     use it.
   - Otherwise: `git worktree add ~/.autopilot/worktrees/<repo>/<id-lowercase>
     -b <branch> origin/<base>` and work from that directory.
   - If the branch already exists in another worktree (`git worktree
     list`), that's a previous run: resume there (step 0).
3. Set it up: install dependencies with the lockfile (`pnpm install
   --frozen-lockfile`, etc.; package stores are shared, so it's fast), and
   copy the gitignored local files the project config lists under
   "Worktree setup" (e.g. `.env.local`) from the main checkout. Copy, never
   symlink, and never commit them.
4. Every later command (servers, tests, captures, git) runs in this
   worktree. Record its path in `.autopilot/state.md`.
5. Native apps (Expo / React Native): a fresh worktree has no native
   build cache, so don't rebuild unless you must. Rebuild the dev client
   only when the change touches native inputs (native dependencies in
   `package.json`, app config, config plugins, native modules or targets,
   `ios/` / `android/`, Podfile). Otherwise reuse the dev client already
   installed on the simulator and only run Metro from this worktree, on a
   free port (step 2b), opening the app at that bundler URL. Confirm the
   app shows this worktree's change before capturing anything.
- **Evidence never gets committed.** All captures, recordings, rendered
  diagrams, logs, and browser state go under `.autopilot/` (captures in
  `.autopilot/captures/`). Before creating anything there, make sure git
  ignores it locally without touching tracked files:

  ```bash
  exclude="$(git rev-parse --git-common-dir)/info/exclude"
  grep -qxF '.autopilot/' "$exclude" || echo '.autopilot/' >> "$exclude"
  ```

  Never write evidence anywhere else in the repo (not `captures/`,
  `test-results/`, `screenshots/`, the repo root). Tool output that lands
  elsewhere by default (Playwright `test-results/`, `playwright-report/`)
  must already be gitignored; if it isn't, add it to the same exclude file.

## 2b. Local servers

Other agents and worktrees may be running dev servers at the same time, so
never assume a port belongs to this branch.

- Get the URL, never guess it: use the dev URL from project config, or the
  URL the dev command prints once it's ready (Next.js `Local:`, Vite,
  portless, Expo). Don't hardcode `localhost:3000`.
- Before starting a server, check whether its port is taken
  (`lsof -iTCP:<port> -sTCP:LISTEN`). If it is and you didn't start it,
  don't use it and don't kill it: start yours on a free port (`PORT=<n>`
  or the framework's `--port` flag) and use that URL.
- Never test or capture against a server you didn't start in this run.
  After starting one, confirm it serves this checkout (e.g. the change is
  visible, or the process's cwd is this worktree) before trusting it.
- Test runners that reuse a running server (Playwright
  `reuseExistingServer`, Cypress `baseUrl`) can silently hit another
  worktree's app. When the default port is busy, point the runner at your
  own server (base URL env var or CLI flag) or run it with `CI=1` where
  that disables reuse, and say which in the PR.
- Stop every server, Metro bundler, and simulator you started before the
  final report. Leave everyone else's alone.

## 3. Plan verification and evidence

Classify the change, then pick **every** row that applies. Decide this
*before* touching code, because "before" evidence must be captured on the
unchanged base.

| Change type | Verification (proof it works) | Evidence (proof for reviewers) |
|---|---|---|
| Visual web UI | Run the app; check desktop + mobile viewports; light/dark if supported | Before/after screenshots via `agent-browser`, published with `before-and-after` |
| Interaction / animation / multi-step UX | E2E test of the interaction when the project has an E2E setup (Playwright, Cypress, Maestro, XCUITest) | Screen recording (before/after when the old behavior exists) |
| iOS / native app | Build and run on the iOS Simulator; XCTest/XCUITest when present | Simulator screenshots or recording, before/after |
| Flow / state machine / architecture / data flow | Unit or integration tests over each branch of the flow | Mermaid diagram of the flow (`pretty-mermaid`), before and after |
| Logic / bug fix | Unit test that fails on the base and passes with the fix (write it first) | Test name + failing-then-passing output |
| API / backend | Integration test; real request against the local server | Request/response transcript (redact secrets) |
| Performance | Benchmark or profile on base vs branch | Numbers table, same machine and conditions |
| Refactor (no behavior change) | Existing suite green; add characterization tests where coverage is thin | Test output + explicit "no behavior change" statement |
| Data migration / schema | Run migration up and down locally against a disposable DB | Migration output; note rollback path |

Rules:

- Always run the project's existing typecheck, lint, and test commands.
- Prefer a new automated test over manual checking whenever the project
  has a test setup for that layer. Don't introduce a new test framework
  just for this issue unless the issue asks for it.
- If a row applies but can't be done (no simulator, app needs secrets you
  don't have), say so explicitly in the PR under **Verification gaps**.
  A gap counts against merge-risk; never paper over it.

## 4. Capture "before"

On the unchanged base, for visual/interaction/native rows:

- Web: start the dev server (use the `run` skill if unsure how), then use
  `agent-browser` (load its core instructions with
  `agent-browser skills get core --full`) to screenshot the exact screens
  and states the issue touches, at desktop and mobile widths. For bugs,
  capture the bug reproducing.
- iOS: in T3 Code, use the device tools (`device_list` → `device_open` →
  its `agent-device` command); elsewhere use `xcrun simctl io booted
  screenshot` / `recordVideo`.
- Save to `.autopilot/captures/` with whitespace-free names:
  `desktop-before.png`, `mobile-before.png`, `flow-before.mp4`.
- For flow rows, write the "before" Mermaid diagram (with `pretty-mermaid`,
  see step 7) from reading the current code now, while it is still the base.

## 5. Implement

- Smallest change that satisfies the acceptance criteria. Fix the case
  that happened, not every case that could. Follow the
  repo's conventions and the relevant skills for the stack (frontend,
  animation, security, etc.).
- Tests first for logic and bug fixes.
- No drive-by refactors. If you notice unrelated problems, list them as
  follow-ups in the PR (optionally create Linear issues for them).
- Commit in logical steps with messages that reference the issue
  (`ABC-123: <summary>`).
- **Stage explicit paths only** (`git add <file>…`, never `git add -A` /
  `git add .`). Before every commit, check `git diff --cached --name-only`:
  nothing under `.autopilot/`, and no image, video, `.har`, `.zip`, trace,
  or browser-state file unless the issue itself adds that asset to the
  product (and then only in the product's asset folder).

## 6. Verify and capture "after"

1. Run everything from step 3: typecheck, lint, unit, E2E, simulator.
   Fix until green. Re-check every acceptance criterion one by one.
2. Capture "after" evidence with the identical viewport, state, and data
   as "before" (same screens, same order, equal heights per
   `before-and-after`).
3. Look at every capture yourself (read the image). If it doesn't clearly
   show the change, recapture.

## 6b. Independent review (sol-loop)

The implementer doesn't grade its own work. Run the `sol-loop` skill on
the local change before pushing, with these autopilot limits that
override its defaults:

- **Budget: at most 2 rounds**, no extension to 5. Stop after round 1 if
  it returns only nits.
- Sol runs as a separate delegated reviewer (sol-loop's `delegate_task`
  route in T3; its native route elsewhere). It sees only the brief and
  the diff. You answer findings and make the fixes yourself.
- Include in the brief: the Linear issue's goal and acceptance criteria,
  the verification already run, and the evidence list.
- After fixes, re-run the affected checks from step 6 (and recapture any
  evidence the fixes changed).
- If a blocker or should is still open after round 2, or Sol can't be
  reached: continue to open the PR, but **auto-merge is off** for this run.
  List the open findings in the PR under **Review findings open**.
- Record the ledger in `.autopilot/state.md` and add sol-loop's line to
  the PR body (`Reviewed with sol-loop: <N> rounds with <model>. …`).

## 7. Open the PR

Before pushing, `git fetch origin` and rebase onto the latest
`origin/<base>` (re-run the step 6 checks if the rebase pulled in
changes), so CI runs on what will merge. Also check that no other open or
merged PR already solves this issue (`gh pr list --search "<issue-id>"`);
if one does, ask the user (see "Asking the user"). Then check the whole
branch once more:
`git diff --name-only origin/<base>...HEAD` must contain no evidence
files (same rule as step 5). Then push and create the PR with
`gh pr create` (never draft unless the `draft` flag was given). Body
template:

```markdown
Fixes ABC-123

## Summary
<1–3 sentences: what changed and why, in user terms>

<!-- before-and-after block goes here -->

## Flow           <!-- only when a flow row applied -->
<mermaid before/after>

## Verification
- [x] <command> — <result>
- [x] <new test name> — fails on base, passes here
- [x] <manual/simulator check>

Reviewed with sol-loop: <N> rounds with <model>. Sol verdict: <verdict>

## Migrations            <!-- omit when none; see step 9b -->
## Verification gaps     <!-- omit when none -->
## Review findings open  <!-- omit when none -->
## Assumptions           <!-- omit when none -->
## Follow-ups            <!-- omit when none -->

<!-- autopilot:state step=7 fix-attempts=0 review-rounds=<n> -->
```

- `Fixes ABC-123` lets Linear auto-close the issue on merge (`Refs` for a
  partial slice, see step 1).
- Attach screenshots/videos with the `before-and-after` skill (its
  `scripts/format.mjs` lives in that skill's directory; run it from the
  repo root so `.autopilot/captures/` paths resolve). Follow its
  video-table workflow for recordings. Uploads go to GitHub's attachment
  storage, never into the repo.
- Diagrams: write them with the `pretty-mermaid` skill (diagram type,
  syntax), render each one locally into `.autopilot/` and look at the
  result, fixing crowding, clipped labels, or parse errors. Post the
  **source** in fenced ```` ```mermaid ```` blocks, not the rendered
  SVG/PNG, so GitHub draws it in the reader's light or dark theme and it
  stays editable. Keep each diagram to ~15 nodes (split otherwise), use
  the same node names in before and after, and mark what changed with
  `classDef changed stroke-width:3px,stroke-dasharray:5 3` (stroke, not
  fill colour, so it reads in both themes). New nodes get the class in
  "after"; removed ones get it in "before".
- Open the rendered PR and confirm media and diagrams display.
- Comment the PR link on the Linear issue and move it to "In Review".
- In T3 Code, call `link_pull_request` with the PR URL.

## 8. CI

- Wait for checks. In T3 Code, use `watch_pull_request` and end the turn
  (you'll be re-invoked; resume from the state file); elsewhere
  `gh pr checks <n> --watch`.
- **Green means every check reported, completed, and passed**, required
  or not. Pending, queued, skipped-with-error, or missing (a check that
  ran on recent PRs but hasn't appeared) is not green.
- On failure: read the failing job's logs (`gh run view --log-failed`),
  fix the real cause, push. Max 3 fix attempts in total across steps 8,
  8b, and 9. Never skip, disable, or weaken a check or test to get green.
  A flaky test may be re-run once; if it fails again, treat it as real.
- If still red after 3 attempts, stop and hand off (step 11).
- **No CI** (project config says `CI: none`, with a local gate): green
  means the config's local gate commands passed on the exact head SHA
  you'll merge, run from a clean tree (`git status --porcelain` empty).
  Post the SHA, commands, and result summary as a PR comment. After any
  push or branch update, run the gate again. Any check that does appear
  on the PR must still be green. Without branch protection, the step 10
  stale-branch checks and `--match-head-commit` are the only guard.
  Never skip them.

## 8b. Preview check

Run when the PR gets a Vercel Preview deployment; skip (and say so in the
PR) when it doesn't. Project config sets the depth; the default is `full`.

1. Find the Preview URL: the Vercel check or bot comment on the PR
   (`gh pr checks <n>`, `gh pr view <n> --comments`), or the Vercel MCP
   (`list_deployments` for the branch). Wait for it to be Ready like any
   other check. A failed Vercel build is a CI failure (step 8).
2. If it's protected (401/403 or a Vercel login page), follow
   `agent-browser skills get protected-vercel-deployments --full`. Never
   put bypass tokens or OIDC tokens in the PR, captures, or logs.
3. With `agent-browser`, open every page the issue touches, at mobile and
   desktop widths:
   - no console errors or failed requests that aren't on production too,
   - each acceptance criterion holds,
   - the main interaction works (`full` depth only; `read-only` depth
     loads and inspects pages but never submits forms, signs in, or
     triggers writes).
4. Evidence: when production is reachable, prefer production as "before"
   and the Preview as "after" (same build type), retaking the captures
   from step 6 and replacing them in the PR's before-and-after block.
   Add the Preview URL to the PR's **Verification** list.
5. A real failure here blocks merge: fix, push, and redo CI and this step
   (counts toward the 3 fix attempts). Something that also fails on
   production is a pre-existing issue: note it as a follow-up.

## 9. Late reviews

Review bots (Vercel Agent, Copilot, CodeRabbit, …) often comment after CI
turns green. Once CI and the Preview are green:

- If a review bot has a pending check or a requested review on the PR,
  wait for it (up to ~10 minutes).
- Read all review comments and threads (`gh pr view <n> --comments`,
  `gh api repos/{owner}/{repo}/pulls/<n>/comments`). For each: fix it
  (smallest change, counts toward fix attempts, back to step 8), or reply
  with the reason it doesn't apply and resolve it. A human's comment or
  requested changes always means hand off, not merge.
- **Bot escalations are not yours to dismiss.** A bot comment or review
  that asks for a human (e.g. "needs human review", "requires manual
  review", a high-risk or security flag with no concrete fix) is not a
  code finding. Fix any concrete issue it names, but leave the
  escalation itself unresolved and list it in `.autopilot/state.md`. The
  merge-risk subagent decides it in step 10.

## 9b. Migrations

Migrations are easy to miss in a PR, and the user applies them by hand,
usually **before** merging. Never apply a migration to production or any
shared database yourself; your job is to find them, check them, and make
sure the user can't miss them.

1. **Detect.** List every migration the branch adds or changes:
   `git diff --name-only origin/<base>...HEAD` against the project
   config's migration paths, or the usual ones (`supabase/migrations/`,
   `prisma/migrations/`, `drizzle/`, `migrations/`, `db/migrate/`,
   `alembic/versions/`). Also check whether the ORM schema changed
   (`schema.prisma`, Drizzle schema files) without a matching migration;
   that's a bug to fix, not to report. No migrations: skip to step 10.
1b. **Check for automatic apply.** Find out whether something already
   applies migrations on merge or deploy: project config first, then
   `.github/workflows/` (and other CI config) for steps like `supabase db
   push`, `supabase migration up`, `prisma migrate deploy`, `drizzle-kit
   migrate`, `knex migrate:latest`, `rails db:migrate`, `alembic upgrade`;
   the build or deploy scripts in `package.json`, `vercel.json` /
   `vercel.ts` (e.g. `prisma migrate deploy && next build`); and platform
   integrations (Supabase GitHub integration, Neon/Vercel). Note *when*
   it runs (before the new code deploys, in parallel with it, or after)
   and whether it needs a manual approval (e.g. a protected GitHub
   environment). If it's automatic and needs no approval, the order is
   fixed by that setup: step 3 checks that this order is safe, and step 6
   doesn't ask you to apply anything.
2. **Check production's state, read-only**, when there's a way (project
   config, Supabase MCP `list_migrations` / `list_tables`, Neon MCP
   `describe_table_schema`): which migrations are already applied, whether
   older migrations from *other* PRs are still pending (call those out,
   they must go first), and the size of the tables touched.
3. **Assess safety in both orders.** Production keeps running the old code
   until the merge deploys, so:
   - *Before merge:* does the **old** code still work against the **new**
     schema? Additive changes are usually safe: new tables, nullable
     columns or columns with a default, new indexes, functions, views.
     Drops, renames, type changes, new `NOT NULL` without a default, or
     tighter constraints/RLS that the old code violates are not.
   - *After merge:* does the **new** code work against the **old** schema
     during the deploy window?
   - *Cost:* locks and rewrites on large tables (index without
     `CONCURRENTLY`, column type change, backfills), duration, statements
     that can't run in a transaction.
   - *Security:* RLS, grants, and policies that widen access.
   - *Rollback:* is there a down migration or a written manual rollback,
     and does it lose data?
4. **Verdict**, one of:
   - **Applied automatically** (step 1b found an automatic apply and its
     order is safe per step 3). If that order isn't safe (e.g. the
     workflow migrates after the deploy, but the new code needs the new
     schema), treat it like "not safe in either order" below.
   - **Run before merge** (the default when safe: matches how the user
     works).
   - **Run after merge** (new code tolerates the old schema, but the old
     code would break on the new one).
   - **Not safe in either order**: restructure into backward-compatible
     steps (expand now, contract in a follow-up issue) when that's a small
     change, then re-run checks from step 6; otherwise ask the user (see
     "Asking the user").
5. **Write it in the PR** under **Migrations**: each file, what it does in
   one line, the verdict and why, lock/duration risk, rollback, pending
   migrations from other PRs, and the exact command to apply it.
6. **Applied automatically:** don't ask. Say so in the PR's
   **Migrations** section, naming the workflow or script that applies
   it, then continue to step 10. After the merge (step 10b), confirm the
   apply ran and succeeded (the workflow run or deploy logs, plus the
   read-only check from step 2 when possible). A failed or missing apply
   is a regression: tell the user right away with the question tool, and
   don't revert automatically (step 10b's rule for migrations).
   **Otherwise, ask the user**, once CI, the Preview, and late reviews are green (so
   it is the last thing between the PR and the merge), with the question
   tool (see "Asking the user"). Include the PR link, the migration files,
   the verdict with its reason in plain words, the exact command, and the
   rollback. Options, recommended first:
   - verdict *before merge*: "I applied it, continue to merge" / "Leave
     the PR open, I'll apply it later" / "Don't apply, it needs changes";
   - verdict *after merge*: "Merge now, I'll apply it right after" /
     "Leave the PR open".
7. **After "applied"**, verify it read-only when you can (step 2's
   method) before merging; if you can't verify, say so in the PR. After
   an after-merge verdict, merge, then remind the user in the final
   report and wait for their confirmation before step 10b's checks.

Record the verdict and the user's answer in `.autopilot/state.md` and the
PR body marker (`migrations=<none|pending|applied|auto>`).

## 10. Merge-risk and merge

**Assess in a fresh subagent.** Spawn a subagent (Claude Code: `Agent`,
foreground; elsewhere the harness's subagent tool) that has none of the
implementation context. Give it only: the PR URL, the Linear issue ID,
and the instruction to run the `merge-risk` skill by reading the PR body,
diff, evidence, checks, review comments, and the issue. Also pass the
list of bot escalations from step 9 (links only, no commentary of
yours), and tell it to rule on each one: **justified** (it points at a
real risk: a sensitive area, a ⚠️-level gap, something the evidence
doesn't cover) or **not justified**, with a one-line reason. Post its
output verbatim as a PR comment. Then reply to each escalation thread
with the subagent's ruling, linking that comment, and resolve the ones
ruled not justified.

Auto-merge only when **all** are true:

- Risk is **low** (or **medium** with the `merge medium` flag).
- No ⚠️ verdict in any category.
- sol-loop finished with no open blocker or should (step 6b).
- No bot escalation (step 9) ruled justified by the merge-risk subagent.
- No migrations, migrations applied automatically in a safe order (step
  9b), or the user answered step 9b's question: applied (and
  verified when possible) for a before-merge verdict, or "merge now" for
  an after-merge one.
- Every check green (step 8); Preview check passed or was skipped for a
  stated reason; no unresolved review threads; no human review comments
  or requested changes.
- No `no merge` / `draft` flag, and the project config doesn't forbid it
  for the touched paths.
- None of these were touched (always hand off, whatever the score): auth
  or permissions, payments, secrets/credentials, PII handling, destructive
  or irreversible data migrations, production infra/deploy config,
  dependency major-version bumps, **any new dependency**.

**Never merge a stale branch.** Immediately before merging:

1. `git fetch origin` and check the PR is up to date with its base
   (`gh pr view <n> --json mergeStateStatus,headRefOid`; `git merge-base
   --is-ancestor origin/<base> HEAD`).
2. If the base moved: update the branch (`gh pr update-branch <n>`, or
   rebase and `git push --force-with-lease` on this PR's own branch),
   then go back to step 8 (CI, Preview, late reviews). If the update had
   conflicts or changed behaviour, re-run sol-loop on the delta (counts
   toward its 2 rounds) and re-assess merge-risk.
3. Pick the merge method: project config first; otherwise the methods the
   repo allows (`gh repo view --json
   squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed`) and how
   recent PRs were merged (`gh pr list --state merged --limit 5`).
4. Merge pinned to the exact commit that was verified, so nothing newer
   slips in:
   `gh pr merge <n> --<method> --delete-branch --match-head-commit <sha>`.
   If GitHub rejects it as out of date, go back to 2. If the repo uses a
   merge queue or auto-merge (`gh pr merge` queues it or enables
   auto-merge instead of merging), keep watching until GitHub reports the
   PR as merged; if it gets dequeued or the queue's checks fail, treat
   that as a CI failure (step 8).

Merge through branch protection as configured. Confirm the merge landed (`gh pr view <n> --json state,mergedAt,
mergeCommit`). Otherwise do **not** merge: leave the PR open, ready for
review, and skip step 10b.

## 10b. Monitor production

After a merge, check production before declaring success. Project config
names the production URLs and any extra signals.

1. Wait for the production deployment of the merge commit (Vercel MCP
   `list_deployments` / `get_deployment`, or the commit's deployment
   status) to be Ready. A failed production build is a regression.
2. Public smoke check with `agent-browser`: load the pages the issue
   touched, plus the home page; no console errors or failed requests.
3. Logs: Vercel `get_runtime_errors` / `get_runtime_logs` for the project,
   for ~15 minutes after the deploy, compared with the hour before it.
   Add any signals the project config lists (e.g. Supabase MCP `query_logs` and `get_advisors`).
4. Logged-in check, only when the project config defines a production
   test account: load its saved browser state from outside the repo
   (path from project config), visit the touched pages read-only, and
   never create, edit, or delete data. If the saved session has expired,
   skip this step and tell the user to refresh it; never type credentials.
5. **On a regression caused by this PR:** open a revert PR immediately
   (`git revert <merge-commit>` on a new branch, `Reverts #<n>` in the
   body, link the evidence), let its CI run, and merge it when green
   (reverting to the last good state is the low-risk move). Reopen the
   Linear issue with a comment explaining what broke. Never revert a PR
   with a migration automatically; tell the user instead (those never
   auto-merge anyway).
6. A problem that already existed before the deploy is not a regression:
   note it as a follow-up.

## 11. Report

End with a short report to the user:

- PR link, CI state, merged or not, production status after merge.
- **Migrations first, when the PR has any:** the files, the verdict
  (before/after merge), whether they're applied, and the exact command
  if not. Never let this get buried lower in the report.
- Merge-risk level and, if not merged, the specific reason(s) and what a
  human should look at.
- sol-loop rounds and outcome; verification run, evidence attached, any
  gaps or assumptions.
- Follow-ups created.
- Each decision you made for the user with a default instead of asking
  (one line each), so they can review it.

Worktree cleanup: once the PR is merged and step 10b passed, stop
everything running from the worktree, then remove it (`git worktree
remove <path>`, or the harness's worktree exit/cleanup tool) and delete
the local branch. Keep the worktree when the PR is left open or a revert
happened, and say where it is in the report.

In T3 Code, `unwatch_pull_request` before the final handback, and before
any earlier stop (hand off, question, blocked issue) on a PR you watch.

## Asking the user

A question in a normal message goes unseen, and the thread looks done. To
ask something, use the harness's question tool so the thread shows it is
waiting: `AskUserQuestion` in Claude Code, `request_user_input` in Codex,
or whatever question tool the harness has. Make it stand on its own: the
issue ID, the PR link if there is one, what blocks you, and the options
with your recommendation first.

Ask only when you cannot continue alone:

- The issue is blocked, ambiguous in a way that changes the result, or
  should be split (step 1).
- You cannot prove the change, and the gap would block the merge.
- The merge gate cannot pass, and you cannot fix that yourself (including
  after 3 fix attempts).
- The next step is outside this skill's grant: deleting data, touching
  production (other than the step 10b revert), changing things outside
  the repo, closing a PR, or force-pushing any branch other than this
  PR's own.
- Another PR makes this one obsolete.

Decide everything else yourself with sensible defaults, and list those
decisions in the PR (**Assumptions**) and the report (step 11).

If no question tool is available, or it returns no answer, take the safe
path: leave the PR open and unmerged, record the question in
`.autopilot/state.md`, and start your final message with the question.

## Never

- Merge with any check red, pending, or missing, with `--admin`, or by
  bypassing branch protection in any other way.
- Merge a branch that isn't up to date with its base, or a head commit
  other than the one verified.
- Force-push to the base branch, or rewrite history others depend on.
- Commit captures, recordings, `.autopilot/`, `.env` files, tokens, or
  browser state files.
- Publish screenshots that show secrets, tokens, or real user PII.
- Use the user's personal browser sessions or type passwords.
- Mark the Linear issue Done yourself before the PR is merged.
