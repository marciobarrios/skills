---
name: open-pr
description: Open a pull request with a consistent title and body (problem-first summary, verification, before/after media, diagrams, follow-ups). Use when the user asks to open, file, create, or ship a PR, and when another skill (such as autopilot) needs to file one.
---

# open-pr

Every PR gets the same shape, whether you open it by hand or `autopilot`
does. A regular PR is just an autopilot PR without the autopilot-only
parts.

## 1. Before filing

1. **Existing PR.** Check whether this branch already has a PR
   (`gh pr view --json url,state`). If it does, update that PR's title
   and body instead of opening another one.
2. **Duplicates.** If the work belongs to a Linear issue, search for other
   open or merged PRs that already solve it (`gh pr list --state all
   --search "<issue-id>"`). If one does, ask the user before going on.
3. **Fresh base.** `git fetch origin`, then rebase onto the latest
   `origin/<base>` so CI runs on what will merge. Only rebase this PR's
   own branch, and only force-push it with `--force-with-lease`. If the
   rebase pulled in changes, re-run the checks you list under
   **Verification**.
4. **Review the diff** against the base (`git diff origin/<base>...HEAD`).
   It must match the goal, with no stray edits or debug code.
5. **No evidence files in the branch.** `git diff --name-only
   origin/<base>...HEAD` must not include `.autopilot/`, captures,
   recordings, `.har`, `.zip`, traces, browser-state files, or `.env`
   files, unless the change itself adds that asset to the product (and
   then only in the product's asset folder). Remove anything else from
   the branch before pushing.

## 2. Title

PR titles usually become commit messages, so follow the repo's title
convention (look at recently merged PRs and `git log`). Keep it short and
readable, and say why the change matters, not how it works:

> ❌ perf(server): negotiate permessage-deflate on the websocket
>
> ✅ perf(server): cut websocket frame size by 70%+ with gzipping

## 3. Body

Use this template. Leave out every section marked optional when it has
nothing in it. Never leave a heading empty or write "N/A".

```markdown
<issue line>                <!-- optional, see below -->

## Summary
<1–3 sentences: the problem, in the user's words, then the fix>

<!-- before-and-after block -->   <!-- optional -->

## Flow                     <!-- optional -->
<mermaid before/after>

## Verification
- [x] <command> — <result>
- [x] <new test name> — fails on base, passes here
- [x] <manual / browser / simulator check>

<review line>               <!-- optional, autopilot only -->

## Migrations               <!-- optional -->
## Verification gaps        <!-- optional -->
## Review findings open     <!-- optional, autopilot only -->
## Assumptions              <!-- optional -->
## Follow-ups               <!-- optional -->

Created with <model> in <harness>.

<state marker>              <!-- optional, autopilot only -->
```

**Issue line.** `Fixes ABC-123` when the PR completes a Linear issue,
`Refs ABC-123` when it's a partial slice. Take the issue from the user,
the caller, or the branch name (Linear branches start with the
lowercased ID, e.g. `abc-123-…`). With no issue, leave the line out;
autopilot's task mode passes `Task: <request>` instead.

**Summary.** Start with a plain explanation of the problem, based on what
the user asked for, then say briefly how it's fixed. Don't lead with an
inventory of what changed:

> ❌ Removed implicit workspace carry-over from every "new thread" entry
> point (cmd+n / cmd+shift+o, sidebar v1/v2 buttons, command palette).
> Deleted buildContextualThreadOptions, startNewThreadInProjectFromContext,
> and the v1 sidebar's seed-context machinery.
>
> ✅ My "new worktree" default was ignored when starting new threads on
> existing worktrees. Super unintuitive. Now your preferences always
> apply.

**Verification.** List only what actually ran on this branch, with its
result. If nothing has run yet in this session, run the project's
typecheck, lint, and the tests covering the changed files first. A check
you couldn't run goes under **Verification gaps**, with the reason. Never
list a check as passed that you didn't see pass.

**Media.** When screenshots or recordings exist (autopilot keeps them in
`.autopilot/captures/`), attach them with the `before-and-after` skill.
Run its `scripts/format.mjs` (in that skill's directory) from the repo
root so the paths resolve, and use its video-table workflow for
recordings. Uploads go to GitHub's attachment storage, never into the
repo. If the user asks for screenshots and none exist, capture them with
`agent-browser` into `.autopilot/captures/`, first adding `.autopilot/`
to `$(git rev-parse --git-common-dir)/info/exclude`. Look at every image
before attaching it.

**Diagrams.** For a change to a flow, state machine, or data flow, write
before and after diagrams with the `pretty-mermaid` skill. Render each
one locally into `.autopilot/` and look at it; fix crowding, clipped
labels, or parse errors. Post the **source** in ```` ```mermaid ````
blocks, not the rendered image, so GitHub draws it in the reader's theme
and it stays editable. Keep each diagram to about 15 nodes (split it
otherwise), use the same node names in before and after, and mark what
changed with `classDef changed stroke-width:3px,stroke-dasharray:5 3`
(new nodes in "after", removed nodes in "before").

**Follow-ups.** Unrelated problems you noticed, and extras you declined
to keep the PR small. One line each.

**Credit line.** End with the public model name and harness that made the
changes, e.g. `Created with <model> in <harness>.` Include subagent models
that wrote part of the change. Use the model name the harness reports; if
it's unavailable, name the harness without guessing a model.

**Autopilot-only parts.** The review line (`Reviewed with sol-loop: …`),
**Review findings open**, and the `<!-- autopilot:state … -->` marker come
from the caller. Put them where the template shows; don't add them
yourself on a regular PR.

## 4. File it

1. Check the title, body, quotes, and attachments for secrets, tokens,
   real user PII, and private details.
2. Push the branch and open a real PR, not a draft, so review bots run.
   Open a draft only when the user or caller asks for one.
3. Write the body to a file and pass it with `--body-file`. If `gh pr
   create` fails because the GraphQL rate limit is used up, open it
   through REST: `gh api repos/<owner>/<repo>/pulls -f title=... -f
   head=... -f base=... -F body=@<file>`.
4. When the body has media or diagrams, open the PR page with
   `agent-browser` and confirm they display.
5. In T3 Code, call `link_pull_request` with the PR URL.
6. If the PR has a Linear issue and the caller isn't handling Linear,
   comment the PR link on the issue.

Don't post a follow-up PR comment unless it serves the request. When a
comment is needed, use the `leaving-pr-comments` skill. If the user also
asked to watch or babysit the PR, continue with the `babysit-pr` skill.

---

The title, summary, credit, and filing rules are adapted from the
`file-pr` skill in [t3dotgg/tstack](https://github.com/t3dotgg/tstack)
(MIT, Copyright (c) 2026 Theo Browne).
