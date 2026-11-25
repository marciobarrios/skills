# Personal Skills

My agent skills for Claude Code, Codex, Cursor and other agents that support
[skills](https://github.com/vercel-labs/skills).

## Install

```bash
npx skills@latest add marciobarrios/skills -g
```

`-g` installs globally (`~/.agents/skills`), so the skills are available in
every project, not only the current one. Add `--skill <name>` to install just
one.

Update later with:

```bash
npx skills@latest update -g
```

## Skills

| Skill | What it does |
|---|---|
| [autopilot](skills/autopilot/SKILL.md) | `autopilot <Linear issue>` or `autopilot <small task>`: implements the issue (or a small task with no issue) end to end, verifies it, gets an independent review, opens a PR with evidence, checks migrations, assesses merge risk, auto-merges when safe, and monitors production. |
| [open-pr](skills/open-pr/SKILL.md) | Opens a PR with a consistent title and body: problem-first summary with a small visual, before/after evidence, verification, merge danger (one-way or two-way door, blast radius), follow-ups. Follows the repo's PR template when there is one. autopilot files its PRs with it, so hand-made and autopilot PRs look the same. |
| [capture-evidence](skills/capture-evidence/SKILL.md) | Plans and captures before/after evidence for a change (screenshots, recordings, Mermaid flow diagrams, failing-then-passing tests), capturing "before" on the base without touching your working tree. autopilot uses it for every run. |
| [check-migrations](skills/check-migrations/SKILL.md) | Finds a PR's database migrations and checks them: automatic apply, production's state (read-only), safety before and after merge, locks, security, rollback. Gives a verdict and the exact command; never applies anything. |
| [safe-merge](skills/safe-merge/SKILL.md) | Merges a PR only when it's safe: merge-risk in a fresh subagent, a strict gate (all checks green, no open threads, no sensitive areas, migrations handled), no stale branches, merged pinned to the verified commit. |
| [vercel-watch](skills/vercel-watch/SKILL.md) | Checks a Vercel deployment for a change. Preview mode checks a PR's Preview in the browser before merge; Production mode watches the deploy after merge (smoke check, runtime errors and logs vs. before) and flags regressions. |

## Requirements

autopilot and the skills it uses call other skills. Install them once per machine:

```bash
npx skills@latest add t3dotgg/tstack --skill sol-loop babysit-pr leaving-pr-comments modernize -g
npx skills@latest add warpdotdev/common-skills --skill merge-risk -g
npx skills@latest add vercel-labs/before-and-after -g
npx skills@latest add vercel-labs/agent-browser -g
npx skills@latest add imxv/Pretty-mermaid-skills -g
```

It also expects the GitHub CLI (`gh`), `python3` (for babysit-pr's `pr-wait`), and, for issue mode, a Linear MCP server.
