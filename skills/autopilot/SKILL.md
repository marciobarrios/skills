---
name: autopilot
description: Use when the user's message starts with "autopilot" followed by a Linear issue URL or ID (e.g. "autopilot https://linear.app/acme/issue/ABC-123/..." or "autopilot ABC-123"), or by a plain description of a small task with no issue (e.g. "autopilot fix the typo in the pricing page footer"). Implements the issue or task end to end, verifies it, gets an independent review, attaches evidence to a PR, assesses merge risk, auto-merges when it is safe, and monitors production afterwards.
---

# autopilot

Take one Linear issue (or a small task with no issue) from "Todo" to a
merged, monitored PR without hand-holding:

read → plan → capture *before* → implement → simplify → verify →
capture *after* →
review (sol-loop) → PR → CI → Preview → merge-risk → merge or hand off →
monitor production.

## Issue mode and task mode

- **Issue mode**: the first argument is a Linear issue URL or an
  identifier like `ABC-123`. Everything below applies as written.
- **Task mode**: anything else. The text after `autopilot` (up to the
  flags) is the task. There is no Linear issue, and none gets created.
  Run the same pipeline with the same gates (verification, evidence,
  sol-loop, CI, Preview, merge-risk, monitoring); only the Linear parts
  change:
  - Skip every Linear read and write: fetching, comments, status
    changes, assigning, `Fixes`/`Refs` lines, reopening.
  - The **task id** is a short kebab-case slug of the task (≤ 5 words,
    e.g. `fix-pricing-footer-typo`). Use it wherever this file says
    "issue ID" for names: branch `autopilot/<slug>`, worktree folder,
    state file, `gh pr list --search` (search the slug and key words of
    the task).
  - Commit messages are plain summaries (`Fix typo in pricing footer`),
    following the repo's commit style; no issue prefix.
  - Anywhere this file says to comment on, or ask via, the Linear issue,
    ask the user instead (see "Asking the user") or put it in the PR.
  - Follow-ups go in the PR's **Follow-ups** section only; don't create
    Linear issues for them.

Arguments after the issue or task are flags in plain words. Honor them:

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
  `<!-- autopilot:state step=<step> fix-attempts=<n> review-rounds=<n> -->`
  (task mode adds `task=<slug>`).

When invoked for an issue that already has an open autopilot PR (or a
worktree on its branch, found with `git worktree list`; in task mode,
an `autopilot/*` branch or PR whose `task=` marker or summary matches the
same task), work in that
worktree, read both, verify them against reality (`gh pr view`, `git
log`), and continue from the recorded step.

If that PR has fallen behind its base (`mergeStateStatus` BEHIND or
DIRTY, or new base commits touch its files), run the `modernize` skill on
it before continuing. Its verdict decides what happens next:
**MODERNIZE**: it rebases and pushes, then continue from step 6 (re-verify
on the new base). **REWORK**: rework it within the issue's scope, then
continue from step 6. **CLOSE**: don't close it yourself; ask the user
(see "Asking the user") with modernize's evidence.

## 1. Read and vet the issue

**Task mode:** there's nothing to fetch. From the user's message (and
the code it points at), write down the goal, acceptance criteria, and
out of scope, then apply items 6 and 7 below with the user in place of
the Linear issue. Task mode is for small fixes: if the size check says
it should be split, or it's clearly more than a small, self-contained
change, ask the user whether to continue anyway or to create a Linear
issue first. Then go to step 2.

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
   Task mode: `autopilot/<slug>`.
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
  diagrams, logs, and browser state go under `.autopilot/`, following the
  `capture-evidence` skill's "Where evidence lives" rules (including the
  local git exclude).

## 2b. Local servers

Every dev server, Metro bundler, or simulator you start in this run
follows `capture-evidence`'s "Local servers" rules: never assume a port
is yours, never test against a server you didn't start, and stop yours
before the final report. If you pointed a test runner at your own server
because the default port was busy, say how in the PR.

## 3. Plan verification and evidence

Plan with the `capture-evidence` skill's table, picking **every** row
that applies. Decide this *before* touching code, because "before"
evidence must be captured on the unchanged base.

- Always run the project's existing typecheck, lint, and test commands.
- Don't introduce a new test framework unless the issue asks for it.
- A row that can't be done goes in the PR under **Verification gaps**.
  A gap counts against merge-risk.

## 4. Capture "before"

Capture "before" with `capture-evidence` now, in this worktree, while it
is still the unchanged base (its "change not written yet" case).

## 5. Implement

- Smallest change that satisfies the acceptance criteria. Fix the case
  that happened, not every case that could. Follow the
  repo's conventions and the relevant skills for the stack (frontend,
  animation, security, etc.).
- Tests first for logic and bug fixes.
- No drive-by refactors: don't clean up code the change doesn't need.
  Edits outside the diff that serve the change (see the simplify pass
  below) are fine. If you notice unrelated problems, list them as
  follow-ups in the PR (optionally create Linear issues for them).
- After the change works, run the `simplify` skill once on this branch's
  diff. Edits outside the diff are fine when they serve the change
  (reusing or extracting helpers/constants, adjusting callers of what you
  consolidated). Cleanups of pre-existing code the change doesn't need
  become follow-ups. If an edit changes behaviour or signatures of
  existing shared code, its other callers must be covered by tests (add
  them if missing) or it becomes a follow-up. Skip for config/copy-only
  or trivial diffs. Commit as `ABC-123: simplify`; step 6 re-verifies.
- Commit in logical steps with messages that reference the issue
  (`ABC-123: <summary>`).
- **Stage explicit paths only** (`git add <file>…`, never `git add -A` /
  `git add .`). Before every commit, check `git diff --cached --name-only`:
  nothing under `.autopilot/`, and no image, video, `.har`, `.zip`, trace,
  or browser-state file unless the issue itself adds that asset to the
  product (and then only in the product's asset folder).

## 6. Verify and capture "after"

1. Run everything from step 3: typecheck, lint, unit, E2E, simulator.
   When it's unclear how to launch the app to check the change, use the
   `run` skill. Fix until green. Re-check every acceptance criterion one by one.
2. Capture "after" with `capture-evidence`: identical viewport, state,
   data, and order as "before", and look at every capture yourself.

## 6b. Independent review (sol-loop)

The implementer doesn't grade its own work. Run the `sol-loop` skill on
the local change before pushing, with these autopilot limits that
override its defaults:

- **Budget: at most 3 rounds**, no extension to 5. Stop after round 1 if
  it returns only nits.
- **Model: always use the latest GPT Sol** offered by
  `orchestrator_capabilities` (sol-loop's default). Don't pin an older
  version. Record which one ran in the ledger.
- Sol runs as a separate delegated reviewer (sol-loop's `delegate_task`
  route in T3; its native route elsewhere). It sees only the brief and
  the diff. You answer findings and make the fixes yourself.
- Include in the brief: the Linear issue's (or task's) goal and acceptance criteria,
  the verification already run, and the evidence list.
- After fixes, re-run the affected checks from step 6 (and recapture any
  evidence the fixes changed).
- If a blocker or should is still open after round 3, or Sol can't be
  reached: continue to open the PR, but **auto-merge is off** for this run.
  List the open findings in the PR under **Review findings open**.
- Record the ledger in `.autopilot/state.md` and add sol-loop's line to
  the PR body (`Reviewed with sol-loop: <N> rounds with <model>. …`).

## 7. Open the PR

File the PR with the `open-pr` skill. It owns the title, the body
template, media, diagrams, the pre-push checks (rebase onto the latest
base, duplicate-PR search, no evidence files in the branch), the
private-details check, and `link_pull_request`. If the rebase pulls in
changes, re-run the step 6 checks. If it finds another PR that already
solves this issue, ask the user (see "Asking the user").

Pass it the autopilot-only parts:

- **Issue line:** `Fixes ABC-123` (or `Refs ABC-123` for a partial slice,
  see step 1). Task mode: `Task: <the user's request, verbatim>`, so
  reviewers and the merge-risk subagent know what was asked.
- **Review line:** `Reviewed with sol-loop: <N> rounds with <model>. Sol
  verdict: <verdict>`, plus **Review findings open** when step 6b left
  any.
- **Sections** from this run: Verification (step 6 and 8b), Flow (when a
  flow row applied), Migrations (step 9b), Verification gaps,
  Assumptions, Follow-ups.
- **State marker:** `<!-- autopilot:state step=7 fix-attempts=0
  review-rounds=<n> -->`.
- **Draft:** only with the `draft` flag.

Then, in issue mode, comment the PR link on the Linear issue and move it
to "In Review".
- **Every GitHub comment, reply, or review you post** in this run (CI
  gate results, replies to bots, the merge-risk output, revert notes)
  goes through the `leaving-pr-comments` skill: its attribution note
  first, then the comment. For the merge-risk output in step 10, the
  note goes above the verbatim text. Not PR descriptions or commit
  messages.

## 8. CI

- Wait with the `babysit-pr` skill's watcher, one per PR, never a
  hand-written polling or `sleep` loop. In T3 Code that is
  `watch_pull_request`: end the turn and resume from the state file when
  woken. Elsewhere it is babysit-pr's `scripts/pr-wait <n> --until ready`
  (`--until green` when the repo has no Greptile or Macroscope), run with
  the Monitor tool in Claude Code. Restart it after every push. Follow
  babysit-pr's rules for rate limits and handling events. A watcher's
  "ready" or "green" is news, not the gate: check the rules below and
  step 10's gate yourself. Autopilot's rules win over babysit-pr's
  (green means every check, and only step 10 merges).
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

Run the `vercel-watch` skill in Preview mode on the PR, with the issue's
(or task's) acceptance criteria and the pages it touches. If it skips
(no Vercel Preview, or depth `skip`), say so in the PR.

- A failed Preview build is a CI failure (step 8).
- A real failure blocks merge: fix, push, and redo CI and this step
  (counts toward the 3 fix attempts).
- Its production-vs-Preview captures replace step 6's in the PR's
  before-and-after block.
- Pre-existing problems become follow-ups.

## 9. Late reviews

Review bots (Vercel Agent, Copilot, CodeRabbit, …) often comment after CI
turns green. Once CI and the Preview are green:

- If a review bot has a pending check or a requested review on the PR,
  wait for it (up to ~10 minutes).
- Read all review comments and threads (`gh pr view <n> --comments`,
  `gh api repos/{owner}/{repo}/pulls/<n>/comments`), and handle them with
  `babysit-pr`'s review rules: verify each finding against the source,
  act only on what's newer than the latest push, treat CodeRabbit as
  advisory, and keep the PR at the issue's size (decline cheap-looking
  extras with a reason and list them as follow-ups). For each: fix it
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

Run the `check-migrations` skill on the PR. No migrations: skip to step
10. Its rules hold here: never apply a migration yourself, and fix an ORM
schema change that has no matching migration (back to step 6). For a
"not safe in either order" verdict, restructure into backward-compatible
steps when that's a small change (expand now, contract in a follow-up
issue) and re-run from step 6; otherwise ask the user (see "Asking the
user").

- **Applied automatically:** don't ask; continue to step 10. After the
  merge, step 10b confirms the apply ran and succeeded (check-migrations
  step 7). A failed or missing apply is a regression: tell the user right
  away with the question tool, and don't revert automatically.
- **Run before / after merge:** ask the user with `safe-merge`'s
  migration question, once CI, the Preview, and late reviews are green,
  so it's the last thing between the PR and the merge. After "applied",
  verify it read-only when you can and say in the PR if you couldn't.
  After an after-merge verdict, merge, then remind the user in the final
  report and wait for their confirmation before step 10b's checks.

Record the verdict and the user's answer in `.autopilot/state.md` and the
PR body marker (`migrations=<none|pending|applied|auto>`).

## 10. Merge-risk and merge

Before calling the merge step, make sure the PR is up to date with its
base. If it isn't, update the branch (`gh pr update-branch <n>`, or
rebase and `git push --force-with-lease` on this PR's own branch) and go
back to step 8 (CI, Preview, late reviews). If the update had conflicts
or changed behaviour, re-run sol-loop on the delta (counts toward its 3
rounds).

Then run the `safe-merge` skill on the PR. It assesses merge-risk in a
fresh subagent, rules on bot escalations, applies its gate, refuses a
stale branch, and merges pinned to the verified commit. Pass it:

- the Linear issue ID (task mode: the user's request, verbatim, and a
  note that there is no Linear issue),
- the bot escalations from step 9 (links only),
- the threshold: `low`, or `medium` with the `merge medium` flag,
- the migration verdict and the user's answer from step 9b,
- autopilot's extra conditions, each a gate item:
  - sol-loop finished with no open blocker or should (step 6b),
  - the Preview check passed or was skipped for a stated reason (step 8b),
  - no `no merge` / `draft` flag.

With `no merge` / `draft`, still run it for the assessment and the
merge-risk comment, but it won't merge. If safe-merge updates the branch
itself because the base moved again, go back to step 8 before letting it
merge. A dequeued PR or a failed merge-queue check counts as a CI failure
(step 8).

If safe-merge doesn't merge, leave the PR open, ready for review, and
skip step 10b.

## 10b. Monitor production

After a merge, run `vercel-watch` in Production mode on the PR before
declaring success.

**On a regression caused by this PR**, revert right away: this request
is the permission vercel-watch asks for. Open a revert PR (`git revert
<merge-commit>` on a new branch, `Reverts #<n>` in the body, link the
evidence), let its CI run, and merge it when green. Reopen the Linear
issue with a comment explaining what broke (task mode: say what broke in
the revert PR and the report). Never revert a PR with a migration
automatically; tell the user instead (those never auto-merge anyway).

A failed or missing automatic migration apply is a regression too: tell
the user right away with the question tool, and don't revert.

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
