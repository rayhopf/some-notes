# Repository Guidelines

## Project Structure & Module Organization
This repository is a small Markdown knowledge base, not an application codebase. Keep long-form procedural notes in the repository root, for example `openclaw-vps-install-procedure.md`. Store recurring monthly reading notes under `reading-list/` using `YYYY-MM.md` filenames such as `reading-list/2026-04.md`.

There is currently no `src/`, `tests/`, or asset pipeline. If you add a new content category, prefer a dedicated directory with a clear name rather than mixing unrelated notes at the root.

## Build, Test, and Development Commands
There is no build step or local server for this repository. Useful maintenance commands:

- `rg --files` lists all tracked note files quickly.
- `rg "keyword" .` searches for terms across notes before adding duplicates.
- `git diff --stat` reviews the scope of your edits before commit.
- `git log --oneline -5` checks recent commit style and naming patterns.

If you use a Markdown linter locally, run it before opening a PR, but no formatter is required by the repository today.

## Coding Style & Naming Conventions
Write in Markdown with clear heading hierarchy and short sections. Use fenced code blocks with language tags for commands, for example ` ```bash `. Prefer descriptive filenames in lowercase with hyphens, and keep date-based reading lists in `YYYY-MM.md` format.

Use bullets and tables only when they improve scanability. Keep line wrapping and spacing consistent within a file instead of reformatting unrelated sections.

## Testing Guidelines
There is no automated test suite. Validation is manual:

- check Markdown renders cleanly in your editor or preview tool
- verify code blocks are copy-pasteable
- confirm links, dates, and command examples are accurate

For procedural guides, prefer testing commands in a safe environment before documenting them.

## Commit & Pull Request Guidelines
Recent history uses short, imperative commit messages such as `Ignore macOS metadata files` and `Add March and April 2026 reading lists`. Follow that pattern. Keep each commit focused on one note set or documentation change.

Pull requests should include a brief summary, the files changed, and any context needed to review factual updates. Include screenshots only if the change depends on rendered Markdown appearance.
