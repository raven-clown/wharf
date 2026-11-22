# AGENTS.md

Instructions for coding agents working in this repository. Keep this file current: when a command, the layout or a rule changes, update it in the same commit.

## Project

wharf is an OCI container registry in Rust that replaces Harbor in place: one binary, Harbor's `/api/v2.0` as its management API, Distribution's storage layout, and an importer that reads a Harbor's database. The design is in `docs/design.md`; implementation plans are in `docs/plans/`. Read the design section a change touches before writing code.

## Commands

There is no code yet. Task 1 of `docs/plans/2026-11-22-phase-1a-registry-core.md` sets up the workspace and fills in this section.

## Rules

- Follow CONTRIBUTING.md: tests with every change, written first, a failing test first for a bug fix, no network in unit tests, no bare sleeps, every test passing on Linux, macOS and Windows.
- Library crates never print and never read environment variables. Only the `wharf` crate does.
- Blob and link paths follow Distribution's layout exactly (`docs/design.md` section 5.2). Changing them breaks in-place migration from Harbor.
- A difference from Harbor's `/api/v2.0`, token service or webhook payloads is a bug. Compatibility changes come with a fixture of Harbor's real request and response.
- No secret (password, token, robot secret, signing key) appears in logs, errors, metrics labels or fixtures.
- Work on a branch off main named `feat/<topic>`, `fix/<topic>`, `ci/<topic>`, `docs/<topic>` or `chore/<topic>`. Never commit straight to main and never force-push it.
- Commit with Conventional Commits messages. One pull request per plan task, closing that task's issue.
- Every pull request gets labels, a milestone and an assignee.
- Never merge a red pull request. Merge your own green one with `gh pr merge <n> --squash --delete-branch --admin --subject "<pr title> (#<n>)" --body ""` (the explicit subject and empty body keep a co-author trailer off main), then `git switch main && git pull`.
- Update CHANGELOG.md under Unreleased for anything users will notice.
- Never mention AI, assistants or code generation in code, comments, docs, commit messages, pull requests or issues. Never add `Co-Authored-By`, `Generated with` or similar trailers.
- No em-dashes. Keep comments short and about why.
