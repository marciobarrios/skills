---
name: capture-evidence
description: Plan and capture before/after evidence for a change (screenshots, recordings, Mermaid flow diagrams, failing-then-passing tests, request transcripts, benchmarks) so a PR can prove it works. Use when the user asks for before/after screenshots, a recording, a diagram, or proof of a change, and when another skill (such as autopilot) needs evidence for a PR.
---

# capture-evidence

Reviewers trust what they can see. This skill decides what evidence a
change needs, captures "before" on the unchanged base and "after" on the
branch under identical conditions, and leaves the files ready for
`open-pr` to attach.

## 1. Plan

Classify the change and pick **every** row that applies:

| Change type | Verification (proof it works) | Evidence (proof for reviewers) |
|---|---|---|
| Visual web UI | Run the app; check desktop + mobile viewports; light/dark if supported | Before/after screenshots via `agent-browser` |
| Interaction / animation / multi-step UX | E2E test of the interaction when the project has an E2E setup (Playwright, Cypress, Maestro, XCUITest) | Screen recording (before/after when the old behavior exists) |
| iOS / native app | Build and run on the iOS Simulator; XCTest/XCUITest when present | Simulator screenshots or recording, before/after |
| Flow / state machine / architecture / data flow | Unit or integration tests over each branch of the flow | Mermaid diagram of the flow (`pretty-mermaid`), before and after |
| Logic / bug fix | Unit test that fails on the base and passes with the fix (write it first) | Test name + failing-then-passing output |
| API / backend | Integration test; real request against the local server | Request/response transcript (redact secrets) |
| Performance | Benchmark or profile on base vs branch | Numbers table, same machine and conditions |
| Refactor (no behavior change) | Existing suite green; add characterization tests where coverage is thin | Test output + explicit "no behavior change" statement |
| Data migration / schema | Run migration up and down locally against a disposable DB | Migration output; note rollback path |

- Prefer a new automated test over manual checking whenever the project
  has a test setup for that layer. Don't add a new test framework just
  for this.
- If a row applies but can't be done (no simulator, the app needs secrets
  you don't have), record it as a **Verification gap** with the reason.
  Never paper over it.
- When only part of the table matters to the user (e.g. "just
  screenshots"), do that part.

## 2. Where evidence lives

**Evidence never gets committed.** Everything goes under `.autopilot/`
(captures in `.autopilot/captures/`). Before creating anything there,
make sure git ignores it locally without touching tracked files:

```bash
exclude="$(git rev-parse --git-common-dir)/info/exclude"
grep -qxF '.autopilot/' "$exclude" || echo '.autopilot/' >> "$exclude"
```

Never write evidence anywhere else in the repo (not `captures/`,
`test-results/`, `screenshots/`, the repo root). Tool output that lands
elsewhere by default (Playwright `test-results/`, `playwright-report/`)
must already be gitignored; if it isn't, add it to the same exclude file.

Use whitespace-free names that pair up: `desktop-before.png` /
`desktop-after.png`, `mobile-before.png`, `flow-before.mp4`,
`flow-before.mmd`.

## 3. Local servers

Other agents and worktrees may be running dev servers at the same time,
so never assume a port belongs to this branch.

- Get the URL, never guess it: use the dev URL from project config
  (`AUTOPILOT.md` or an `## Autopilot` section in `AGENTS.md` /
  `CLAUDE.md`), or the URL the dev command prints once it's ready
  (Next.js `Local:`, Vite, portless, Expo). Don't hardcode
  `localhost:3000`. Use the `run` skill when unsure how to launch the app.
- Before starting a server, check whether its port is taken
  (`lsof -iTCP:<port> -sTCP:LISTEN`). If it is and you didn't start it,
  don't use it and don't kill it: start yours on a free port (`PORT=<n>`
  or the framework's `--port` flag).
- Never capture against a server you didn't start. After starting one,
  confirm it serves this checkout (the change is visible, or the
  process's cwd is this checkout) before trusting it.
- Test runners that reuse a running server (Playwright
  `reuseExistingServer`, Cypress `baseUrl`) can silently hit another
  worktree's app. When the default port is busy, point the runner at your
  own server (base URL env var or CLI flag) or run it with `CI=1` where
  that disables reuse, and say which.
- Stop every server, Metro bundler, and simulator you started when you're
  done. Leave everyone else's alone.

## 4. Capture "before"

"Before" must come from the unchanged base.

- **Change not written yet** (e.g. autopilot): capture now, in the
  current checkout.
- **Change already on the branch or in the working tree**: don't stash,
  reset, or switch branches. Make a throwaway worktree of the base:
  `git worktree add --detach "$TMPDIR/before-<repo>" origin/<base>`,
  install dependencies with the lockfile, copy the gitignored local files
  the app needs (e.g. `.env.local`; copy, never symlink), run the app
  there on a free port, capture, then stop it and `git worktree remove`
  it. If production shows the base behavior and is reachable, it can be
  "before" instead; say so.

What to capture:

- **Web:** with `agent-browser` (load its core instructions with
  `agent-browser skills get core --full`), screenshot the exact screens
  and states the change touches, at desktop and mobile widths. For a bug,
  capture it reproducing.
- **iOS:** in T3 Code, use the device tools (`device_list` →
  `device_open` → its `agent-device` command); elsewhere `xcrun simctl io
  booted screenshot` / `recordVideo`. In a fresh worktree, reuse the
  installed dev client and only run Metro from that checkout on a free
  port, unless the change touches native inputs.
- **Flow:** write the "before" Mermaid diagram (with `pretty-mermaid`)
  from reading the base code.
- **Logic / API / performance:** run the new test, request, or benchmark
  against the base and keep the output.

## 5. Capture "after"

On the branch, with the identical viewport, state, data, and order as
"before" (equal heights, per `before-and-after`). For flows, write the
"after" diagram with the same node names.

Look at every capture yourself (read the image, watch the recording's
key frames). If it doesn't clearly show the change, recapture. Never
publish a capture that shows secrets, tokens, or real user PII.

## 6. Hand off

Report the list of files in `.autopilot/captures/`, which rows of the
table they cover, and any gaps. To put them on a PR, use `open-pr` (it
attaches media with `before-and-after` and posts diagrams as Mermaid
source).
