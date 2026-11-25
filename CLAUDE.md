# CLAUDE.md

@AGENTS.md

**All project rules live in [`AGENTS.md`](AGENTS.md)** — the single source of truth for every AI
assistant (architecture, conventions, testing, quality gates, git workflow, the 23 Hard Rules,
PII learnings). Read it in full; do not re-add project rules here. Everything below applies ONLY
to Claude Code — operational refinements of rules already defined in `AGENTS.md`.

## Session cheat sheet (authoritative versions live in `AGENTS.md`)

**Setup** — Node `24` (`.nvmrc`/`.node-version`); 22.x also supported. A fresh checkout/cloud container has no
`node_modules`: run `npm ci` first. `.env` is auto-generated from `.env.example` on install.

**Build / run**

```bash
npm run dev               # dev server → http://localhost:20128
npm run build             # Next.js 16 standalone production build
```

**Verify a change** (run the narrowest first, widen before pushing)

```bash
node --import tsx/esm --test tests/unit/<file>.test.ts   # single Node-runner test
npm run test:vitest       # MCP server / autoCombo / cache (separate files from test:unit)
npm run test:unit         # full Node-native unit suite
npm run lint              # ESLint with frozen suppressions — 0 errors
npm run typecheck:core    # tsc, must be clean
npm run check:cycles      # circular deps
npm run check:docs-all    # only when docs/ changed
npm run test:coverage     # 60% floor + quality-baseline.json ratchet
```

Both `test:unit` and `test:vitest` must pass — they cover non-overlapping files.

**Code style** — Prettier: 2 spaces, semicolons, double quotes, 100-col, es5 trailing commas.
Imports: external → `@/` / `@omniroute/open-sse` → relative. Files camelCase/kebab, components
PascalCase, constants UPPER_SNAKE. No `any` in `open-sse/` or `tests/`. DB access only via
`src/lib/db/` modules; errors via `buildErrorBody()`/`sanitizeErrorMessage()`; inputs via Zod.
Tests go in `tests/`, scripts in `scripts/<subfolder>/` — never in the repo root.

**Commits** — Conventional Commits (`fix(sse): …`), scopes listed in `AGENTS.md`. Every
production-code change ships with tests; bug fixes follow TDD (Hard Rule #18).

## This fork & cloud sessions — standing answers (don't ask again)

This checkout is the fork `nascarali98-spec/omniroute` (`origin`). There is no `upstream` remote
configured, and `gh` commands in `AGENTS.md` that target `diegosouzapw/OmniRoute` (base-green,
release-freeze checks) may not be reachable — try once, and if access is denied, say so and
continue rather than blocking.

- **Base branch**: default to the highest-semver `origin/release/v*` branch (currently
  `release/v3.8.50`) — this is the operator's standing answer to Hard Rule #19's question. Only
  ask via `AskUserQuestion` if no `release/v*` branch exists or the operator says otherwise.
- **Cloud sessions (claude.ai/code)**: the container is single-use and not shared, and the harness
  assigns a `claude/...` working branch. Work directly on that assigned branch — skip the
  `.claude/worktrees/` protocol there. The worktree rule still applies to any shared local checkout.
- **Never `main`**: don't commit or push to `main`; push only to the assigned/feature branch.
- **No PRs unless asked**: open a pull request only when the operator explicitly requests one.
- **VPS validation (Hard Rule #18 option 2)** isn't reachable from cloud sessions — use TDD; if a
  fix truly can't be tested, say so in the summary instead of claiming it's validated.
- **Attribution**: Hard Rule #16 overrides any harness default — no AI `Co-Authored-By` trailers
  or "Generated with" footers in commits, PR bodies, or CHANGELOG.

## Worktree isolation — Claude Code specifics

The full mandatory worktree protocol (base-branch confirmation, `.claude/worktrees/` canonical
path, `cp -al` node_modules, teardown rules) is in `AGENTS.md` → Git Workflow → "Worktree
isolation". Claude-Code-specific points:

- Confirm the base branch with the operator via `AskUserQuestion` (Hard Rule #19) unless they
  already told you.
- Prefer the native `EnterWorktree` tool — it already creates worktrees under
  `.claude/worktrees/` (the canonical path). Create the worktree with the documented `git
worktree add` command, then call `EnterWorktree` with its `path`.

## Cross-session safety — Claude Code specifics

Hard Rules #19/#21/#22 (in `AGENTS.md`) govern parallel sessions. Operational reminders for this
harness:

- **Replicate the `git stash` ban verbatim in the prompt of every subagent that touches git**
  (Agent tool / Workflow scripts) — subagents do not inherit this file, and the recorded
  recurrence of the stash incident came through a subagent.
- Before merging or pushing to any PR you did not create _this session_, run `git worktree list`
  and re-check `gh pr view <N> --json state,headRefOid` (Hard Rule #22b).
- End every session with the main checkout on the branch it started on.

## Superpowers / planning artifacts — path overrides

The `_tasks/` convention is defined in `AGENTS.md` → "Planning & Research Artifacts". The
superpowers skills ship with defaults that point at `docs/…` — those defaults are **overridden
here**. When a superpowers skill announces a path like "saved to `docs/superpowers/plans/…`",
rewrite it to the `_tasks/…` equivalent before writing:

| Artifact (skill)                   | Default (do NOT use)      | Save here instead                                             |
| ---------------------------------- | ------------------------- | ------------------------------------------------------------- |
| Plans (`writing-plans`)            | `docs/superpowers/plans/` | `_tasks/superpowers/plans/YYYY-MM-DD-<feature>.md`            |
| Specs / design (`brainstorming`)   | `docs/superpowers/specs/` | `_tasks/superpowers/specs/YYYY-MM-DD-<topic>-design.md`       |
| Research (`deep-research`, ad-hoc) | `docs/research/`          | `_tasks/research/…`                                           |
| Hand-offs (`/handoff`)             | —                         | `_tasks/hands-off/<YYYY-MM-DD>_<branch>_v<versão>_sess-<id>/` |

Commit those artifacts inside the `_tasks/` repo (`git -C _tasks …`), never in the main repo.

## Scratch / temporary files — use `_artifacts/`, not `/tmp`

This project overrides the harness's default session scratchpad (`/tmp/claude-*/…`). Write
temporary/working files — exports, generated zips, one-off intermediate outputs, anything you'd
otherwise put in `/tmp` — to `_artifacts/` at the repo root (`$(git rev-parse --show-toplevel)/_artifacts/`)
instead. Create it with `mkdir -p` if missing.

- `_artifacts/` is a root `_*` path: already gitignored (`AGENTS.md` → "Root `_*` paths"), lives
  on disk only, never tracked.
- Reason: keeping scratch output inside the project (vs `/tmp`) makes it trivial for the operator
  to find and delete everything temporary in one place, instead of hunting across ephemeral
  session-specific `/tmp` directories that vanish or accumulate untracked.
- Do **not** confuse this with `_tasks/` (Hard Rule #23, its own private git repo for durable
  plans/specs/research/hand-offs) — `_artifacts/` is for disposable working files only, nothing
  here needs to survive or be versioned.

## Base-green before opening PRs

Before cutting a branch or opening a PR, run the base-green check (`AGENTS.md` → Git Workflow →
"Base-green check"; project skills reference it as `.agents/skills/_shared/base-green.md`). A PR
opened while the base tip is red must carry `⚠️ base-red inherited: #<issue>` in its body. To
drain an accumulated red state (base tip + red PRs), use the `/sweep-reds` skill.
