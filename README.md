# skills

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

## Adding a skill

Create `skills/<name>/SKILL.md` with `name` and `description` frontmatter, add
it to the table above, and push.
