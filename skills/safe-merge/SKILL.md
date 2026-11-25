---
name: safe-merge
description: Merge a pull request only when it's safe. Assesses merge risk in a fresh subagent (the merge-risk skill), checks a strict gate (all checks green, no open threads or human change requests, no sensitive areas, migrations handled), refuses stale branches, and merges pinned to the verified commit. Use when the user asks to merge a PR "if it's safe", to check whether a PR can be merged, or when another skill (such as autopilot) reaches its merge step.
---

# safe-merge

Merge only what you can defend. When the gate doesn't pass, don't merge:
leave the PR open and say exactly why.

Inputs: the PR (number or URL). Optional, from the user or a caller:

- **Risk threshold:** `low` by default; `medium` only when asked (e.g.
  `merge medium`).
- **Issue or request:** the Linear issue ID, or the request in words when
  there's no issue.
- **Bot escalations:** links to bot comments that ask for a human.
- **Extra conditions** the caller must have met (e.g. autopilot's
  "sol-loop has no open blocker"). Each one is a gate item.

Project config (`AUTOPILOT.md`, or an `## Autopilot` section in
`AGENTS.md` / `CLAUDE.md`) may set the merge method, paths that must
never auto-merge, and extra merge rules. It wins over this file.

Every GitHub comment or reply you post here goes through the
`leaving-pr-comments` skill.

## 1. Read the PR's state

`gh pr view <n> --json
state,isDraft,mergeStateStatus,headRefOid,baseRefName,reviewDecision,statusCheckRollup,files`,
plus review threads (`gh api graphql` for `reviewThreads` with
`isResolved`) and comments.

- **Checks:** every check reported, completed, and passed, required or
  not. Pending, queued, skipped-with-error, or missing (a check that ran
  on recent PRs but hasn't appeared) is not green. If checks are still
  running, wait with the `babysit-pr` skill's watcher (in T3 Code,
  `watch_pull_request`), never a `sleep` loop.
- **Bot escalations:** if the caller didn't pass them, collect bot
  comments or reviews that ask for a human ("needs human review", a
  high-risk or security flag with no concrete fix). Links only.

## 2. Migrations

If the PR adds or changes migrations, run `check-migrations` unless the
caller already has its verdict. The gate needs one of:

- verdict **applied automatically**,
- verdict **run before merge**, and the user confirmed it's applied
  (verified read-only when possible),
- verdict **run after merge**, and the user said to merge now.

Otherwise ask the user with the harness's question tool
(`AskUserQuestion` in Claude Code): the PR link, the files, the verdict
in plain words, the exact command, and the rollback. Options, recommended
first: for *before merge*, "I applied it, continue to merge" / "Leave the
PR open, I'll apply it later" / "Don't apply, it needs changes"; for
*after merge*, "Merge now, I'll apply it right after" / "Leave the PR
open".

## 3. Assess in a fresh subagent

The one who wrote or watched the change doesn't grade it. Spawn a
subagent (Claude Code: `Agent`, foreground; elsewhere the harness's
subagent tool) with none of your context. Give it only:

- the PR URL,
- the Linear issue ID, or the request verbatim with a note that there is
  no issue,
- the bot escalation links, with no commentary of yours,
- the instruction to run the `merge-risk` skill by reading the PR body,
  diff, evidence, checks, review comments, and the issue, and to rule on
  each escalation: **justified** (it points at a real risk: a sensitive
  area, a ⚠️-level gap, something the evidence doesn't cover) or **not
  justified**, with a one-line reason,
- and to check the PR body's **Merge danger** claim (door and blast
  radius) against the diff: **agree** or **disagree**, with the door and
  blast radius it would state instead. A missing claim counts as
  disagree.

Post its output verbatim as a PR comment (the attribution note above it).
Then reply to each escalation thread with the subagent's ruling, linking
that comment, and resolve the ones ruled not justified.

## 4. The gate

Merge only when **all** are true:

- Risk is at or below the threshold (`low`, or `medium` when allowed).
- No ⚠️ verdict in any category.
- No bot escalation ruled justified.
- The change is a **two-way door**, and the subagent agrees with the
  PR's Merge danger claim. A one-way door, or a claim it disputes,
  always hands off.
- Migrations handled (step 2).
- Every check green (step 1).
- No unresolved review threads; no human review comments or requested
  changes still open.
- Not a draft; the project config doesn't forbid it for the touched
  paths; every extra condition from the caller holds.
- None of these were touched (always hand off, whatever the score): auth
  or permissions, payments, secrets/credentials, PII handling,
  destructive or irreversible data migrations, production infra/deploy
  config, dependency major-version bumps, **any new dependency**.

If any item fails, stop: don't merge, and report each failing item and
what a human should look at.

## 5. Never merge a stale branch

Immediately before merging:

1. `git fetch origin` and check the PR is up to date with its base
   (`mergeStateStatus`; `git merge-base --is-ancestor origin/<base>
   <head-sha>`).
2. If the base moved: update the branch (`gh pr update-branch <n>`, or
   rebase and `git push --force-with-lease` on this PR's own branch,
   never anyone else's). Then go back to step 1: wait for checks again.
   If the update had conflicts or changed behaviour, re-run step 3 on the
   new head.

## 6. Merge

1. Pick the method: project config first; otherwise what the repo allows
   (`gh repo view --json
   squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed`) and how
   recent PRs were merged (`gh pr list --state merged --limit 5`).
2. Merge pinned to the exact commit that passed the gate, so nothing
   newer slips in:
   `gh pr merge <n> --<method> --delete-branch --match-head-commit <sha>`.
   If GitHub rejects it as out of date, go back to step 5.
3. If the repo uses a merge queue or auto-merge (`gh pr merge` queues it
   or enables auto-merge), keep watching until GitHub reports the PR as
   merged. If it gets dequeued or the queue's checks fail, it's not
   merged: report it.
4. Confirm it landed: `gh pr view <n> --json state,mergedAt,mergeCommit`.

## Report

Merged or not, the merge commit, the risk level, and, if not merged, each
failing gate item. When the PR has migrations, put them first: the
verdict, whether they're applied, and the exact command if not.

## Never

- Merge with any check red, pending, or missing, with `--admin`, or by
  bypassing branch protection in any other way.
- Merge a branch that isn't up to date with its base, or a head commit
  other than the one that passed the gate.
- Force-push any branch other than this PR's own, or rewrite history
  others depend on.
- Apply a migration yourself.
