# pull-all Sync Redesign

## Tracking Issue

https://forge.anarch.diy/allod/tools/issues/87

This is a three-PR implementation in `allod/tools`. PR 1 and PR 2 should use
`Refs allod/tools#87`; PR 3 should use `Closes allod/tools#87` after the
explicit reset mode lands.

## Goal

Make `pull-all` a fast, live workspace sync command that is safe by default and
explicit when destructive reset or clean behavior is requested.

## Scope

In scope:

- `workspace/pull-all` CLI parsing, help text, concurrency, output, and Git sync
  behavior.
- `tests/workspace/pull-all.sh` fixture coverage for normal sync, `--switch`,
  retry handling, and destructive reset/clean modes.
- `workspace/README.md` documentation for user-facing `pull-all` behavior.

Out of scope:

- Nix, system lifecycle, provisioning, VM activation, or wrapper generation.
- `work-diff`, `allod change`, or other workspace tools.
- Running destructive reset or clean operations against the real workspace during
  implementation validation.

## Risk Assessment

Residual risk: R4 Critical for the full issue, because PR 3 introduces an
explicit destructive reset and clean mode.

Why:

- PR 1 and PR 2 change a frequently used Git workspace command across every repo
  under `~/work`.
- PR 2 reduces default risk by replacing `git pull` with fetch plus
  fast-forward-only sync, but it still changes branch-state decisions and output
  users rely on.
- PR 3 intentionally adds operations that can discard local commits, index
  state, worktree changes, untracked files, and ignored files when requested.
- Fixture tests can cover command selection and reporting, but human review
  should still inspect safety boundaries and help text before merge.

Human scrutiny:

- Confirm each PR only changes the intended CLI contract, tests, and README.
- Review skip/reset decisions, option incompatibilities, and error messages
  before trying new behavior on a real workspace.
- For PR 3, inspect the ref validation and clean/reset ordering first.

| PR or milestone | Risk | Reason | Human scrutiny |
|---|---|---|---|
| PR 1: live parallel `git pull` | R2 Medium | Restores faster parallel output and adds `--jobs`, while keeping existing `git pull` semantics. | Verify concurrency parsing, live output readability, and help/tests. |
| PR 2: fetch plus fast-forward-only sync | R3 High | Changes default sync from `git pull` to explicit fetch/status/ff-only behavior across repos. | Verify ahead/diverged/no-upstream handling, retry behavior, and safe `--switch`. |
| PR 3: explicit reset/clean mode | R4 Critical | Adds intentionally destructive reset and clean options. | Verify remote-tracking ref restriction, option incompatibilities, warnings, and fixture-only validation. |

## Interface Contracts

Shared contracts:

- The command remains `pull-all`.
- Normal mode must never run `git reset --hard`, `git clean`, or any equivalent
  destructive operation.
- Dirty working trees are skipped in normal mode.
- Existing safe `--switch` behavior remains: a non-default branch is switched to
  the default branch only when the working tree is clean and the current branch
  has no unpushed or diverged work relative to its upstream.
- Worker output may be emitted in completion order once live output is added, but
  each status line must identify the repo.

PR 1 contracts:

- Add `--jobs N` as the preferred concurrency flag.
- Keep `PULL_ALL_JOBS=N` as a compatibility fallback.
- `--jobs N` takes precedence over `PULL_ALL_JOBS`.
- `N` must be a positive integer; invalid values fail before syncing starts.
- Raise the default concurrency from `4` to `8`.
- Keep using `git pull` for sync in this PR.
- Help text must document `--jobs N`, `PULL_ALL_JOBS`, `--switch`, `-h/--help`,
  the default job count, and that PR 1 still uses `git pull`.

PR 2 contracts:

- Replace normal `git pull` with:
  1. `git fetch` with retry controls.
  2. upstream detection for the branch being synced.
  3. ahead/behind comparison against the upstream.
  4. fast-forward-only update when the branch is only behind.
- Add `--fetch-retries N`, where `N` is a non-negative integer. Default: `1`
  retry after the initial fetch attempt.
- Add `--fetch-retry-delay SECONDS`, where `SECONDS` is a non-negative integer.
  Default: `1`.
- A branch with no upstream is reported clearly and skipped in normal mode.
- A branch that is ahead of upstream is reported clearly and skipped.
- A branch that has both local and upstream-only commits is reported as diverged
  and skipped.
- A branch that is only behind upstream is advanced with fast-forward-only
  semantics and reported as updated.
- `--switch` continues to skip local-only, ahead, or diverged work branches
  rather than checking them out or rewriting them.
- Help text must document the fetch retry flags, fast-forward-only behavior,
  ahead/diverged/no-upstream reporting, and the preserved `--switch` safety rule.

PR 3 contracts:

- Add `--reset-to REF` as the only entry point for destructive reset mode.
- Initial support accepts only remote-tracking refs, such as
  `--reset-to origin/master` and `--reset-to origin/HEAD`.
- Reject local branches, tags, raw commit IDs, and missing refs for
  `--reset-to`.
- In reset mode, fetch first, validate `REF` after fetch, then run
  `git reset --hard REF` for each selected repo.
- Add `--clean`, valid only with `--reset-to`; it runs `git clean -fd` after a
  successful reset.
- Add `--clean-ignored`, valid only with `--reset-to`; it runs `git clean -fdx`
  after a successful reset and implies `--clean`.
- Normal sync mode must reject `--clean` and `--clean-ignored`.
- The initial PR should reject combining `--reset-to` with `--switch`; the reset
  target already defines the desired commit.
- Help text must clearly label `--reset-to`, `--clean`, and `--clean-ignored` as
  destructive, state that only remote-tracking refs are accepted initially, and
  include examples using `--reset-to origin/master` and
  `--reset-to origin/HEAD`.

## Agent Gates

- Do not run `pull-all --reset-to`, `pull-all --clean`, or
  `pull-all --clean-ignored` against the real `/home/allod/work` workspace during
  implementation. Validate those paths only through test fixtures.
- Do not broaden this into Nix, provisioning, lifecycle, secrets, or host-only
  work. This plan covers CLI behavior in `allod/tools`.

## Acceptance Tests

```sh
cd /home/allod/work/allod/tools
bash -n workspace/pull-all tests/workspace/pull-all.sh
bash tests/workspace/pull-all.sh
workspace/pull-all --help
```

Required fixture coverage:

- PR 1 tests assert `--jobs` parsing, `--jobs` precedence over `PULL_ALL_JOBS`,
  invalid job rejection, default job count behavior, live output before all jobs
  finish, existing `git pull` calls, and updated help text.
- PR 2 tests assert fetch retries, exhausted fetch failure reporting,
  fast-forward-only update, up-to-date reporting, ahead skip, diverged skip,
  no-upstream skip, dirty skip, and safe `--switch` behavior.
- PR 3 tests assert `--reset-to origin/master` and `--reset-to origin/HEAD`
  acceptance, rejection of non-remote refs, reset command ordering after fetch,
  `--clean` and `--clean-ignored` command selection, rejection of clean flags
  without `--reset-to`, rejection of `--reset-to` with `--switch`, and
  destructive help text with the required examples.

## Rollback Plan

- Before merge, abandon the implementation branch or worktree.
- After merge, revert the affected PR in `allod/tools`. PRs are intentionally
  ordered so PR 1 can be reverted independently, PR 2 can restore prior
  `git pull` behavior, and PR 3 can remove destructive reset/clean mode without
  changing normal sync contracts.
- If a normal-mode regression affects real repos, inspect `git status`, branch
  upstreams, and `git reflog` in each affected repo before applying manual Git
  repair.
- If reset/clean mode is misused on real repos, deleted untracked or ignored
  files may not be recoverable from Git; rely on backups or external copies, and
  use `git reflog` only for tracked commits and branch tips.
