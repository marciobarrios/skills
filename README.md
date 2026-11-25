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
| [open-pr](skills/open-pr/SKILL.md) | Opens a PR with a consistent title and body: problem-first summary, verification, before/after media, diagrams, follow-ups. autopilot files its PRs with it, so hand-made and autopilot PRs look the same. |

## Requirements

autopilot and open-pr call other skills. Install them once per machine:

```bash
npx skills@latest add t3dotgg/tstack --skill sol-loop babysit-pr leaving-pr-comments modernize -g
npx skills@latest add warpdotdev/common-skills --skill merge-risk -g
npx skills@latest add vercel-labs/before-and-after -g
npx skills@latest add vercel-labs/agent-browser -g
npx skills@latest add imxv/Pretty-mermaid-skills -g
```

It also expects the GitHub CLI (`gh`), `python3` (for babysit-pr's `pr-wait`), and, for issue mode, a Linear MCP server.
