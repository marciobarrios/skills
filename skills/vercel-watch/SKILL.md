---
name: vercel-watch
description: Check a Vercel deployment for a change. Preview mode checks a PR's Preview deployment in the browser (pages load, no new console errors or failed requests, acceptance criteria hold). Production mode watches the production deploy after a merge (build Ready, smoke check, runtime errors and logs compared with before the deploy) and flags regressions. Use when the user asks to check a Preview, verify a deploy, or watch production after a merge, and when another skill (such as autopilot) needs those checks.
---

# vercel-watch

Two modes, one job: prove the deployed change works, and catch it early
when it doesn't.

- **Preview:** a PR's Preview deployment, before merge.
- **Production:** the production deployment of a merge commit, after
  merge.

Pick the mode from the request: an open PR means Preview, a merged PR or
"after the deploy" means Production. Inputs: the PR (number or URL), or
a deployment URL, plus what the change is meant to do (issue, acceptance
criteria, or the request in words) and the pages it touches. When no
pages are given, derive them from the diff (routes, components, and the
pages that use them).

Project config (`AUTOPILOT.md`, or an `## Autopilot` section in
`AGENTS.md` / `CLAUDE.md`) may set: the Preview check depth (`full`,
`read-only`, or `skip`; default `full`), production URLs, extra signals
to watch, and a production test account. It wins over this file.

Use `agent-browser` for every browser step (load its core instructions
with `agent-browser skills get core --full`). Use the Vercel MCP for
deployments and logs, with the project and team IDs from
`.vercel/project.json` when present.

## Preview mode

Skip, and say so, when the PR gets no Vercel Preview deployment or the
depth is `skip`.

1. **Find the Preview URL:** the Vercel check or bot comment on the PR
   (`gh pr checks <n>`, `gh pr view <n> --comments`), or the Vercel MCP
   (`list_deployments` for the branch). Wait for it to be Ready with the
   `babysit-pr` watcher (in T3 Code, `watch_pull_request`), never a
   `sleep` loop. A failed build is a failure: read the build logs
   (`get_deployment` and its events) and report the cause.
2. **Protected?** On a 401/403 or a Vercel login page, follow
   `agent-browser skills get protected-vercel-deployments --full`. Never
   put bypass tokens or OIDC tokens in the PR, captures, or logs.
3. **Check every page the change touches**, at mobile and desktop widths:
   - no console errors or failed requests that aren't on production too,
   - each acceptance criterion holds,
   - the main interaction works (`full` depth only; `read-only` depth
     loads and inspects pages but never submits forms, signs in, or
     triggers writes).
4. **Evidence:** when production is reachable, use production as "before"
   and the Preview as "after" (same build type), with `capture-evidence`'s
   rules (identical viewports, files in `.autopilot/captures/`). Add the
   Preview URL to the PR's **Verification** list when you can edit the
   PR, keeping the rest of the body unchanged.
5. **Verdict:** pass, or fail with each failure and its evidence.
   Something that also fails on production is pre-existing, not a
   failure: report it as a follow-up.

## Production mode

1. **Wait for the deploy** of the merge commit (Vercel MCP
   `list_deployments` / `get_deployment`, or the commit's deployment
   status) to be Ready. A failed production build is a regression.
2. **Smoke check:** load the pages the change touched, plus the home
   page; no console errors or failed requests.
3. **Logs:** Vercel `get_runtime_errors` / `get_runtime_logs` for the
   project, for about 15 minutes after the deploy, compared with the hour
   before it. New errors, or a clear rise in existing ones, on the touched
   routes are a regression. Add any signals the project config lists
   (e.g. Supabase MCP `query_logs` and `get_advisors`).
4. **Logged-in check,** only when the project config defines a production
   test account: load its saved browser state from outside the repo (path
   from project config), visit the touched pages read-only, and never
   create, edit, or delete data. If the saved session has expired, skip
   this and tell the user to refresh it. Never type credentials.
5. **Migrations:** if the PR's migrations were set to apply automatically,
   confirm the apply ran and succeeded (`check-migrations` step 7).
6. **Verdict:** healthy, or a regression with its evidence (the failing
   page, error, or log lines, and the deploy). A problem that already
   existed before the deploy is not a regression: report it as a
   follow-up.

On a regression, don't revert on your own: report it first, and
recommend a revert PR (`git revert <merge-commit>`, `Reverts #<n>`) as
the low-risk move. Open it only when the user or caller asks. Never
revert a PR with a migration automatically.

## Never

- Write to production data, sign in with real user accounts, or type
  passwords.
- Use the user's personal browser sessions.
- Publish captures that show secrets, tokens, or real user PII.
