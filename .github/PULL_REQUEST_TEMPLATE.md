<!--
Thanks for the pull request. The comments in this template are hidden when it renders, so you can leave them or delete them.

TITLE
Use Conventional Commits: `type(scope): what changes`, in lowercase and the imperative.
  fix(oci): return 416 for a range past the end of a blob
  feat(harbor): list robot accounts of a project
  docs(design): describe the importer cutover
Types: feat, fix, docs, test, refactor, perf, build, ci, chore. Add `!` after the type for a breaking change.
The title becomes the commit on main, because pull requests are squash merged.

HOW TO WRITE THE DESCRIPTION
- Lead with what changes for a user, a client or an operator, then say why.
- Plain English, short sentences, active voice.
- Show before and after when it helps: a request and response, a config snippet, a log line, in a code block.
- Name files, crates, flags, config keys and endpoints in backticks.
- State facts. If you measured something, give the number and how you measured it.

If there is an issue, put `Closes #123` on the first line. If there is none, delete that line.

CHECKS
Workflows from a fork start once a maintainer approves them, so the checks can show as waiting for a while. That's expected.
-->

Closes #

## What changes

## Why

## How it works

## How I tested it

<!-- The tests you added and what each one proves. For a bug fix, say that the new test fails on main and passes here. -->

## What I didn't test

<!-- Platforms, storage backends or clients you couldn't try. "Nothing" is a fine answer. -->

## wharf checks

<!-- Tick what applies and delete the rest. -->

- [ ] Blob and link paths still follow Distribution's layout exactly
- [ ] A `/api/v2.0` change matches Harbor's request and response, with a fixture from a real Harbor
- [ ] A schema change adds a migration and keeps rollback to the previous minor release possible
- [ ] No password, token, robot secret or key reaches logs, errors, metric labels or fixtures
- [ ] A permission change has tests for both allow and deny
- [ ] A new config key works from TOML and from its `WHARF_` variable, and `docs/` documents it

## Checklist

- [ ] Tests cover the change, and a bug fix has a test that fails without it
- [ ] `cargo fmt`, `cargo clippy -- -D warnings` and `cargo test --workspace` pass
- [ ] `CHANGELOG.md` has a line under Unreleased, if users will notice the change
- [ ] Docs are updated, if usage, config or commands changed
