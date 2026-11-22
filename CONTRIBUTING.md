# Contributing

Thanks for helping out. Bug reports, compatibility reports, docs fixes, tests and new features are all welcome.

## Finding something to work on

- Issues labeled `good first issue` are small and say exactly what to change.
- Issues labeled `help wanted` are bigger. Each one links the plan task it comes from, with the interface and the tests it needs.
- Milestones group issues by phase. Start with the earliest open one.
- The [wharf board](https://github.com/users/raven-clown/projects/8) tracks every plan task and who has it.

If you take an issue, leave a comment on it so nobody else picks it up. If you go quiet for two weeks, someone else may take it over.

The design is in [`docs/design.md`](docs/design.md) and the implementation plans are in [`docs/plans/`](docs/plans/). Read the design section your change touches before you start.

## You don't need an issue first

- **Straight to a pull request:** bug fixes, docs, tests, a fix for a Harbor compatibility gap, and anything else small enough to review in one sitting.
- **A short design comment first:** a new config key, a schema change, a new crate, a change to how a feature behaves next to Harbor, or a change that touches many crates. Write a few lines on what you want to do and how it looks to a user, in an issue or at the top of a draft pull request. A maintainer answers before you spend days on the code.

The issue forms are there when you want to report something without fixing it: a bug, a Harbor compatibility gap, a feature request or a docs problem.

## Development setup

You need the Rust toolchain pinned in `rust-toolchain.toml` (rustup installs it on first build).

```shell
git clone https://github.com/<your-username>/wharf.git
cd wharf

cargo build
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all
```

The conformance suite and the client end-to-end tests run on Linux: `bash conformance/run.sh` and `bash e2e/run.sh`. They need Go and Docker.

## Tests

Every change comes with tests. A pull request without them is only merged when there is nothing to test, such as a typo.

- A bug fix starts with a test that fails without the fix.
- A feature is covered for the normal case, the error cases and the edges the issue lists.
- A Harbor compatibility fix includes the request and the response Harbor gives, as a fixture.
- Test behavior through the public API of a crate. Don't test private helpers unless they hold real logic.
- Tests must not depend on timing. Use `tokio::time::pause`, channels or polling with a deadline, never a bare sleep.
- No network calls in unit tests.
- Every test passes on Linux, macOS and Windows.

## Code style

- `cargo fmt` and `cargo clippy` with no warnings.
- Errors carry context. Library crates return typed errors; only the `wharf` binary crate prints.
- Keep comments short and about why, not what.
- No new dependency without a reason in the pull request. `cargo deny` must pass.
- No `unsafe`.

## Commits and pull requests

- One change per pull request. A refactor and a feature are two pull requests.
- The pull request title follows [Conventional Commits](https://www.conventionalcommits.org/), such as `fix(oci): reject a tag longer than 128 characters` or `feat(harbor): list robot accounts`. Pull requests are squash merged, so the title becomes the commit on `main`.
- If there is an issue, link it with `Closes #123`.
- Describe what changed, why, and how you tested it. Say what you didn't test.
- Keep the branch up to date with `main` and respond to review. A maintainer aims to give a first review within three days.

## How a pull request gets merged

`main` is protected. A pull request is merged only when all of these pass:

- a review approval from a code owner
- every review thread resolved
- a linear history (squash merge)
- every required check green, as listed on the pull request

Workflows from a fork wait for a maintainer to approve them before they run. Dependabot opens weekly pull requests for crates and GitHub Actions, and they go through the same checks.

## Backward compatibility

Until 1.0, a minor release may change config keys and the CLI, and the changelog says how to migrate. The OCI API and the Harbor-compatible API never break on purpose: a difference from Harbor is a bug.

Every user-visible change adds a line to `CHANGELOG.md` under Unreleased.

## Reporting a security problem

Don't open a public issue. Use GitHub's private vulnerability reporting on the repository instead. See [SECURITY.md](SECURITY.md).

## Code of conduct

Be kind and assume good intent. This project follows the [Contributor Covenant](CODE_OF_CONDUCT.md).
