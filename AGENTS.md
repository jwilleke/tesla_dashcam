---
project_state: "active"
last_updated: "2026-07-19"
agent_priority_level: "medium"
blockers: []
requires_human_review: []
agent_autonomy_level: "high"
kit_version: "3aa1bb4"
---

<!-- KIT:START 3aa1bb4 — managed by mjs-project-template; edit below the KIT:END marker -->
# Agent Context & Protocols

This section is **managed by the kit** (`install-kit.sh`) — it is identical across repos. Put repo-specific context **below the `KIT:END` marker**; do not edit here.

## Session continuity

- Before starting, read the `▶ Resume here` block at the top of `TODO.md` (committed, so it syncs across machines) and recent `git log`. That is where the last session left off — repeating finished work is the most common avoidable mistake.
- Commit a chunk of work with `/session-commit`: commits code + `TODO.md`, appends a journal entry to `private/project_log.md` (the log is never committed).
- Run `/pstatus` often (after every `/session-commit`): it ranks open work and recommends the next step.
- End a session with `/wrap`: commits anything outstanding, refreshes the `▶ Resume here` pointer, and reports whether it is safe to shut down the editor.

## Priorities — GitHub labels are the source of truth

Priority labels are mutually exclusive and mean:

- `P0` — **Broken. Stop all work and fix it.** (production down / blocked / security breach)
- `P1` — **Delivers value to the mission.**
- `P2` — **Nice to have.**
- `deferred` — consciously postponed; `needs-triage` — awaiting a priority decision.

Then:

- Security comes first. Scanner alerts (Dependabot / code-scanning / GitGuardian) become issues labeled `security` + a graded priority: critical/high → `P0`, medium → `P1`, low → `P2`.
- `TODO.md` = a `▶ Resume here` block (maintained by `/wrap`) on top, then priority bands that `/pstatus` regenerates from the labels. Do not hand-edit the bands.

## Working agreement

- Think before coding: state assumptions, surface trade-offs, ask when scope is ambiguous.
- Simplicity first: the minimum that solves the problem; nothing speculative.
- Use Conventional Commits for messages.
- Issue decomposition — NEVER put "Steps", "Phases", or numbered sequences inside a single GitHub issue. Break each step into its own issue and link them using GitHub relationships: `closes #N` / `fixes #N` (resolves another), `blocked by #N` (dependency), `relates to #N` (context link). Example: a 3-phase migration = 3 issues with "blocked by" chains, not one issue with Phase headings.
- Issue/PR links — Never use a bare `#N` reference alone. Always pair it with the full GitHub URL: `[#333](https://github.com/owner/repo/issues/333)`. This applies in commit messages, PR descriptions, comments, and any agent output. Use `/issues/N` for issues and `/pull/N` for PRs.
- Awaiting approval — When work is complete but requires human sign-off before closing, apply the `in-review` label and leave a comment on the issue/PR that states: what was done, what the human needs to verify, and what action closes it. Never self-close an issue or PR.
- Commits — always use the `/session-commit` skill. Never run a bare `git commit` directly. `/session-commit` enforces the session log update, conventional commit format, and co-author trailer.

## Markdown conventions

- Dash (`-`) bullets; no bare numbered lists. ATX (`#`) headings. Spaced tables (`| a | b |`).
- Inline HTML is **not** allowed. Long lines are fine.
- Rules live in `.markdownlint.jsonc`; the editor, CLI, CI and agents all read that one file.
<!-- KIT:END -->

# tesla_dashcam

## Project Context

This is my fork of [ehendrix23/tesla_dashcam](https://github.com/ehendrix23/tesla_dashcam)
(`upstream`). `origin` is my fork; I push here and can pull upstream changes, but never push
to upstream. The working branch is `dev`.

`tesla_dashcam` is a Python 3.13+ CLI that merges the per-minute, per-camera MP4 files Tesla
saves for Dashcam/Sentry events into a single movie, using **ffmpeg** as the encoding engine.

- **Full build/run/architecture reference:** see [CLAUDE.md](CLAUDE.md). It is the detailed guide
  (commands, the single-module architecture in `tesla_dashcam/tesla_dashcam.py`, the layout
  classes, and the processing pipeline). Read it before making code changes.
- **Run:** `python -m tesla_dashcam <source> --output <dest>` (needs `ffmpeg` on PATH).
- **Test:** `pytest` (pure unit tests; no ffmpeg or sample footage required).
- **Deps:** `pip install -r requirements.txt` (add `requirements_dev.txt` for pytest/doc8).

### Key decisions for an agent

- Almost the entire program is one ~6,000-line module (`tesla_dashcam/tesla_dashcam.py`). Work
  within it; the split-into-modules TODO at the top of that file is aspirational, not done.
- Version lives in two places kept in sync: the `VERSION` dict in `tesla_dashcam.py` and
  `tesla_dashcam/__version__.py` (the latter is what `setup.py` reads).
- CI upstream is CodeQL only; there is no enforced Python linter. Markdown here is linted via the
  kit's `.markdownlint.jsonc` + `markdown-lint.yml`.
- Behavior changes should be reflected in `README.md` (the authoritative user docs) and, where
  relevant, the `Preference_Files/*.txt` argparse presets.
