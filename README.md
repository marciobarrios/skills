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
| [autopilot](skills/autopilot/SKILL.md) | `autopilot <Linear issue>`: implements the issue end to end, verifies it, gets an independent review, opens a PR with evidence, checks migrations, assesses merge risk, auto-merges when safe, and monitors production. |

## Requirements

autopilot calls other skills. Install them once per machine:

```bash
npx skills@latest add t3dotgg/tstack --skill sol-loop -g
npx skills@latest add warpdotdev/common-skills --skill merge-risk -g
npx skills@latest add vercel-labs/before-and-after -g
npx skills@latest add vercel-labs/agent-browser -g
npx skills@latest add imxv/Pretty-mermaid-skills -g
```

It also expects the GitHub CLI (`gh`) and a Linear MCP server.
