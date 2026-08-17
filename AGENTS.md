# AGENTS.md

## Cursor Cloud specific instructions

### What this repo is
This is a **GitHub profile README repository** (`github.com/Kasyaprk/Kasyaprk`). It is pure
Markdown content — there is **no application, build system, test suite, or CI**. The tracked
files are `README.md` (the profile page GitHub renders on the user's profile),
`docs/ai-engineer-growth-plan.md`, and `.gitignore`. "Developing" means editing Markdown;
"running" means previewing the rendered Markdown as GitHub would display it.

### Dev toolchain (installed by the update script)
There are no codebase dependencies. For the authoring workflow, the update script installs two
CLI helpers:
- `markdownlint-cli2` (via npm, prefix `~/.npm-global`) → binary at `~/.npm-global/bin/markdownlint-cli2`
- `grip` (GitHub Readme Instant Preview, via `pip --user`) → binary at `~/.local/bin/grip`

Non-obvious: this VM boots without a saved snapshot, so `~/.bashrc` / PATH edits do **not**
persist across sessions. If these binaries aren't on your PATH, add:
`export PATH="$HOME/.npm-global/bin:$HOME/.local/bin:$PATH"` (or call them by full path).

### Lint
Run `markdownlint-cli2 "**/*.md"` from the repo root. Note: the default ruleset flags the
profile README's intentional styling (inline `<div align="center">`, badge lines over 80 chars,
emphasis-as-heading, GFM table alignment). These are expected for a GitHub profile README and are
rendered correctly by GitHub — do not "fix" them by rewriting the content unless explicitly asked.

### Build / run (preview)
There is no build. To preview locally the way GitHub renders it:
`grip README.md 0.0.0.0:6419` then open `http://localhost:6419/`.
`grip` re-reads the file on each request, so edit the Markdown and refresh the browser to see
changes. `grip` renders via GitHub's Markdown API (needs network egress; unauthenticated requests
are rate-limited to ~60/hour). Preview `docs/ai-engineer-growth-plan.md` with
`grip docs/ai-engineer-growth-plan.md 0.0.0.0:6420`.

### Tests
There are no automated tests in this repository.
