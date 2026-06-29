# allod patch

## Tracking Issue

[allod/tools#77](https://forge.anarch.diy/allod/tools/issues/77). Single PR against `allod/tools`.

## Goal

Add an `allod patch` subcommand that transfers committed changes from a private-capable dev VM to a public-authorized host as `git format-patch` artifacts, so the human can review and push without granting one environment both capabilities.

## Scope

In scope:

- `allod/tools/allod` — add `patch` namespace with `fetch`, `apply`, `receive` subcommands
- `allod/tools/tests/allod-patch.sh` — test suite
- `allod/tools/docs/allod-patch.md` — usage documentation with examples

Out of scope:

- Packaging changes (nexus, profiles, flake locks) — separate follow-up after the tool lands
- Changes to `allod change` subcommands
- PR body validation or `forge` integration (patch doesn't create PRs)
- Automatic push (always explicit `--push`)

## Risk Assessment

Residual risk: R3 High

Why:

- New SSH-based cross-machine transport — no existing SSH code in the tool to build on. A bug could silently leave artifacts on the remote or fail to transfer patches.
- Privacy/security boundary — the tool's purpose is preventing private context from leaking to public repos. The remote command surface must be minimal and auditable.
- `git am` application can fail in ways that leave the destination repo in a conflicted state. The tool must handle this cleanly.
- Rollback is a straight revert of the tools PR. No persistent state beyond the git commits applied by `apply`.

Human scrutiny:

- The SSH command strings in `fetch` — verify the remote command surface is minimal (only git and checksum operations, no shell expansion of user input).
- Manifest format and checksum verification in `apply` — confirm sha256 comparison is correct.
- Cleanup behavior on SSH failure — verify no orphaned temp dirs on the remote.
- `git am` failure handling — verify the destination worktree is left clean (aborted) on failure.

## Interface Contracts

### CLI

```
allod patch fetch <ssh-host>:<source-repo> [--base <ref>] [--output <dir>]
allod patch apply <artifact-dir> [--repo <destination-repo>] [--push]
allod patch receive <ssh-host>:<source-repo> <destination-repo> [--base <ref>] [--push]
```

`--base` defaults to `origin/master`.

### `patch fetch`

Runs on the public-authorized host. SSHes into the source VM and generates the patch artifact.

1. Parse `<ssh-host>:<source-repo>` into SSH host and absolute repo path.
2. SSH into the source VM and run a single remote script that:
   a. Validates the source worktree is clean (`git diff --quiet && git diff --cached --quiet`). Exits non-zero if dirty.
   b. Resolves `--base` ref. Validates HEAD is ahead of base (`git rev-list <base>..HEAD` is non-empty). Exits non-zero if nothing to export.
   c. Creates a temp dir on the remote.
   d. Runs `git format-patch <base>..HEAD -o <tmpdir>`.
   e. Writes a JSON manifest to `<tmpdir>/manifest.json` containing: `repo_remote` (origin URL), `base_commit` (full SHA), `head_commit` (full SHA), `patch_count`, and `patches` (array of `{filename, sha256}`).
   f. Outputs the temp dir path to stdout.
3. Transfer the artifact directory from the remote to the local `--output` dir (default: `/tmp/allod-patch-<timestamp>-XXXXXX` via mktemp). Use `tar cz -C <tmpdir> . | ssh ... cat` piped in reverse: `ssh <host> "tar cz -C <tmpdir> ."` piped to local `tar xz -C <output>`.
4. Remove the remote temp dir after successful transfer (SSH rm -rf). On transfer failure, print the remote temp dir path so the human can clean up manually.
5. Print summary: patch count, base..head range, local artifact path.

Exit codes:
- `0` — success
- `1` — usage error, SSH failure, or remote error
- `10` — source worktree is dirty
- `11` — no commits to export (HEAD is not ahead of base)

### `patch apply`

Runs on the public-authorized host against the destination repo.

1. Read `manifest.json` from the artifact directory.
2. Verify sha256 of each patch file matches the manifest. Exit non-zero on mismatch.
3. Resolve destination repo (from `--repo` or cwd). Require clean worktree.
4. Verify destination repo's `origin` URL matches `manifest.repo_remote`. Exit non-zero on mismatch with actionable message showing both URLs.
5. Verify `manifest.base_commit` is reachable in the destination repo. Exit non-zero with explanation if not (e.g., "run git fetch first").
6. Apply patches: `git am --3way <artifact-dir>/*.patch`. On failure, run `git am --abort` and exit non-zero.
7. Post-apply checks: `git diff --check <base>..HEAD`, `git log --oneline <base>..HEAD`, `git show --stat --oneline HEAD`.
8. If `--push`: `git push`. Otherwise print reminder.

Exit codes:
- `0` — success
- `1` — usage error
- `12` — checksum mismatch
- `13` — repo identity mismatch (remote URL)
- `14` — base commit not reachable
- `15` — `git am` failed (patches aborted)
- `16` — destination worktree dirty

### `patch receive`

Smooth-path wrapper. Runs `fetch` then `apply`. Passes `--push` through to `apply`. Exits with the first non-zero exit code.

### Manifest format

```json
{
  "repo_remote": "ssh://git@forge.anarch.diy:2222/allod/memory.git",
  "base_commit": "abc123...",
  "head_commit": "def456...",
  "patch_count": 2,
  "patches": [
    {"filename": "0001-some-change.patch", "sha256": "..."},
    {"filename": "0002-another-change.patch", "sha256": "..."}
  ]
}
```

### Exit code summary

```
0   success
1   usage error / SSH failure / general error
10  source worktree dirty
11  no commits to export
12  checksum mismatch
13  repo identity mismatch
14  base commit not reachable
15  git am failed
16  destination worktree dirty
```

These start at 10 to avoid collision with `allod change` exit codes (1-6) since they share the `allod` script.

## Agent Gates

- Human reviews the SSH command surface in `fetch` before merging. The remote command must not pass unsanitized user input to a shell.
- Human tests the cross-VM workflow manually after the tool lands (the automated tests use local SSH mocking, not real VMs).

## Acceptance Tests

Test script: `allod/tools/tests/allod-patch.sh`

### SSH mock strategy

Tests do not use real SSH. Instead, create a mock `ssh` script on PATH that:
- Parses the host and remote command from args
- Executes the remote command locally against a test repo (simulating the remote)
- This works because the remote commands are pure git + coreutils operations

Similarly, mock `scp`/`tar` transfer by having the mock SSH write to a local directory that simulates the remote filesystem.

### fetch tests

- Happy path: single commit ahead of base produces one patch + valid manifest. Verify patch count, filenames, sha256s in manifest, artifact dir exists locally.
- Multiple commits: produces multiple patches in order.
- Dirty source worktree: exits 10 with actionable message.
- No commits to export (HEAD == base): exits 11.
- SSH connection failure (mock returns non-zero): exits 1 with message.
- Remote cleanup: after successful fetch, remote temp dir is removed (verify via mock).
- `--output <dir>`: artifacts land in specified directory.
- `--base` custom ref: patches cover the specified range.

### apply tests

- Happy path: clean destination repo, matching remote URL, reachable base, patches apply cleanly. Verify commits exist after apply.
- Checksum mismatch: tamper with a patch file after fetch. Exits 12.
- Repo identity mismatch: destination has different origin URL. Exits 13 with both URLs shown.
- Base commit not reachable: destination is behind. Exits 14 with "git fetch" hint.
- Dirty destination worktree: exits 16.
- `git am` conflict: create a conflicting commit in destination before apply. Exits 15, worktree is clean after abort.
- `--push`: verify git push is called (via mock git wrapper).
- Without `--push`: verify git push is NOT called.
- Multiple patches: all apply in order, log shows correct range.

### receive tests

- Happy path: end-to-end fetch + apply succeeds. Verify commits in destination.
- Fetch failure propagates: dirty source causes exit 10 (not masked).
- Apply failure propagates: checksum mismatch causes exit 12.
- `--push` passed through to apply.

### Run

```sh
bash allod/tools/tests/allod-patch.sh
```

## Rollback Plan

Revert the commit that adds `patch` namespace to `allod/tools/allod`, the test file, and the docs file. No persistent state, no packaging changes, no flake lock updates in this PR.
