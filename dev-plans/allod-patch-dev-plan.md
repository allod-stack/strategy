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
- Rollback is a straight revert of the tools PR. Validation may create local artifact dirs, intentionally leave a printed remote temp dir after transfer failure, or apply git commits during smoke tests; those states have explicit cleanup steps in the rollback plan.

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

### Top-level dispatch

The existing `allod` script must add a `patch` namespace alongside `change`:

- Top-level `usage` lists both `change` and `patch`.
- `main` routes `allod patch ...` to `patch_main`, and `patch_main` routes only `fetch`, `apply`, `receive`, and `-h|--help`.
- Missing or unknown patch subcommands exit 1 with the patch usage text.
- Patch-specific failures use 10-16. Generic usage/SSH/remote failures may use 1 even though `allod change` also uses 1 for generic failure.

### Remote SSH contract

Every SSH invocation in `patch fetch` must keep user-supplied values out of the remote command string.

- Invoke SSH with the host as an argument (`ssh -- "$host" ...`) after rejecting an empty host, empty source repo, and relative source repo path.
- The remote command text must be static. Do not build commands such as `ssh "$host" "tar cz -C $tmpdir ."` or interpolate `<source-repo>`, `--base`, or the remote temp dir path into remote shell syntax.
- OpenSSH does not preserve remote command argv as argv; arguments after the host are serialized into remote shell command text. Do not pass raw or base64-encoded dynamic values as remote command arguments or environment assignments. Pass `<source-repo>`, `--base`, and later the remote temp dir path through stdin, such as base64 data lines consumed by a static remote bootstrap before it `exec`s a static `bash -se` script. If base64 is used, encode each value as a single line (`base64 -w 0` or equivalent wrapping removal) before transport. Reject decoded values containing newlines; shell arguments cannot carry NUL bytes, so do not add a separate NUL-handling path.
- Quote every decoded value in remote commands (`git -C "$repo" ...`, `tar -cz -C "$tmpdir" .`, `rm -rf -- "$tmpdir"`). Do not use user input in `mktemp` templates.
- Treat the remote temp dir path as untrusted control data after generation. The local fetch side must accept exactly one stdout line and require it to match the fixed `/tmp/allod-patch.XXXXXXXXXX` shape before using it for tar or cleanup; the remote tar and cleanup scripts must repeat the same decoded path validation before `tar -cz -C "$tmpdir" .` or `rm -rf -- "$tmpdir"`.
- Keep remote generation stdout as a control channel. `git format-patch` output must go to stderr or a log file; stdout from the generation command is exactly one line containing the remote temp dir path.

### `patch fetch`

Runs on the public-authorized host. SSHes into the source VM and generates the patch artifact.

1. Parse `<ssh-host>:<source-repo>` into SSH host and absolute repo path.
2. Before any SSH, resolve the final local artifact dir and preflight the local destination. If `--output` is provided, its parent must already exist as a writable directory and the final output path must not exist, including a dangling symlink. Destination preflight failures exit 1 before remote work.
3. SSH into the source VM using the remote SSH contract and run a single remote script that:
   a. Validates the source worktree is clean with `git status --porcelain` empty, so tracked, staged, and untracked changes are all blocked. Exits 10 if dirty.
   b. Resolves `--base` ref to `base_commit` with `git rev-parse --verify "$base^{commit}"`. Validates `base_commit` is an ancestor of `HEAD` with `git merge-base --is-ancestor "$base_commit" HEAD`, then validates `git rev-list "$base_commit..HEAD"` is non-empty. Rejects any merge commit in the export range with `git rev-list --merges "$base_commit..HEAD"` non-empty, because `git format-patch` omits merge commits and would not faithfully represent the source branch. Exits 11 if the source range is not exportable.
   c. Creates a temp dir on the remote with a fixed `mktemp -d /tmp/allod-patch.XXXXXXXXXX` template and installs a trap that removes it if generation fails before the path is handed back.
   d. Runs `git format-patch "$base_commit..HEAD" -o "$tmpdir"` with stdout redirected away from the control channel. Verifies at least one `.patch` file was created in `$tmpdir`; if `git format-patch` produced zero patches (e.g., every commit in the range is empty), exits 11.
   e. Writes a JSON manifest to `<tmpdir>/manifest.json` with `jq`, containing: `repo_remote` (origin URL), `base_commit` (full SHA), `head_commit` (full SHA), `patch_count`, and `patches` (array of `{filename, sha256}`).
   f. Outputs only the temp dir path to stdout.
4. Create a local staging dir with `mktemp -d` in the final dir's parent. Extract into staging; after successful tar extraction and a regular non-symlink `manifest.json`, rename the staging dir to the final output path. For the default output, the `mktemp -d /tmp/allod-patch.XXXXXXXXXX` path may be the final dir, but it must be removed on transfer or validation failure.
5. Transfer the artifact directory using a static SSH tar script that decodes and validates the remote temp dir path, then runs `tar -cz -C "$tmpdir" .`, piped to local `tar -xz -C "$staging"`.
6. Remove the remote temp dir only after local tar extraction succeeds, `manifest.json` is regular and not a symlink, and the staging dir has been moved or accepted as the final output. Use a static SSH cleanup script that decodes and validates the temp dir path before running `rm -rf -- "$tmpdir"`. On transfer, local extraction, or local artifact validation failure, remove the local staging dir, leave the final output absent, and print the remote temp dir path so the human can clean up manually. If cleanup SSH fails after local promotion, keep the local artifact, exit 1, and print both the local artifact path and remote temp dir path for manual cleanup; do not report fetch success.
7. Print summary: patch count, base..head range, local artifact path.

Exit codes:
- `0` — success
- `1` — usage error, SSH failure, or remote error
- `10` — source worktree is dirty
- `11` — source range is not exportable: base is not an ancestor, `HEAD` is not strictly ahead of base, the range contains merge commits, or all commits in the range are empty

### `patch apply`

Runs on the public-authorized host against the destination repo.

1. Resolve the artifact directory to an absolute path. Require `<artifact-dir>/manifest.json` to be a regular non-symlink file before reading it.
2. Validate the manifest JSON shape and verify sha256 of each listed patch file matches the manifest. Exit 12 on malformed JSON, missing or wrong-type required fields, non-full or non-hex `base_commit`/`head_commit`, unsafe filename, missing listed patch file, non-regular listed patch file, patch count mismatch, duplicate filename, unlisted `.patch` file, or checksum mismatch.
3. Resolve destination repo (from `--repo` or cwd). Require clean worktree.
4. Verify destination repo's `origin` URL matches `manifest.repo_remote`. Exit 13 on mismatch with actionable message showing both URLs.
5. Verify `manifest.base_commit` exists and is an ancestor of the current destination `HEAD` with `git merge-base --is-ancestor "$base_commit" HEAD`. Exit 14 with explanation if not (e.g., "run git fetch first" or "check out the destination branch containing the base commit").
6. Record `pre_apply_head=$(git rev-parse HEAD)`. Apply patches in manifest order: build an array by joining each validated `manifest.patches[].filename` basename to the resolved artifact directory, preferably as absolute paths, and run `git am --3way "${patch_files[@]}"`. Do not use bare manifest basenames with `git -C "$repo"` and do not use a shell glob.
7. On `git am` failure, run `git am --abort` when an am state directory exists, then assert `HEAD` is still `pre_apply_head`, `git status --porcelain` is empty, and `$(git rev-parse --git-path rebase-apply)` plus `$(git rev-parse --git-path rebase-merge)` are absent. Exit 15 either way, but if cleanup assertions fail, print the failed assertion and the repo path for manual repair.
8. Post-apply checks use `applied_range="$pre_apply_head..HEAD"`, not `base_commit..HEAD`, so pre-existing destination commits are not revalidated. Run `git diff --check "$applied_range"` before any push. If it fails, leave the applied commits in place, skip push, exit 1, and print the repo path, `pre_apply_head`, current `HEAD`, and manual fix/reset guidance.
9. Print summaries for the applied range: `git log --oneline "$applied_range"` and `git show --stat --oneline HEAD`.
10. If `--push` and post-apply checks passed: run `git push`. If `git push` fails, leave the applied commits in place, exit 1, and print the repo path, `pre_apply_head`, current `HEAD`, and manual push/reset guidance. Otherwise print reminder.

Exit codes:
- `0` — success
- `1` — usage error, post-apply validation failure after commits were applied and push was skipped, or push failure after commits were applied
- `12` — manifest/checksum integrity failure
- `13` — repo identity mismatch (remote URL)
- `14` — base commit missing or not an ancestor of destination HEAD
- `15` — `git am` failed (patches aborted)
- `16` — destination worktree dirty

### `patch receive`

Smooth-path wrapper. It must not parse human-readable `fetch` summary output to discover the artifact path.

1. Create a receive-owned parent temp dir with `mktemp -d /tmp/allod-patch-receive.XXXXXXXXXX`, set `artifact_dir="$parent/artifact"`, and run the same fetch implementation with `--output "$artifact_dir"` so the output path is exact and not pre-existing.
2. If fetch fails before promoting `artifact_dir`, remove the receive-owned parent dir and exit with fetch's status. If fetch fails after `artifact_dir` exists, such as cleanup SSH failure after local promotion, leave the artifact dir in place, print it, exit with fetch's status, and do not run apply.
3. Run apply against `artifact_dir`. Pass `--push` through to `apply`.
4. Leave `artifact_dir` in place and print it on apply failure or success, so the human can inspect the transferred patches. Exits with the first non-zero exit code.

### Manifest format

Use `jq` for manifest generation and parsing. Use `sha256sum` from coreutils for checksums; do not introduce a Python dependency.

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

Checksum and filename rules:

- Compute each digest from the patch file bytes with `sha256sum -b -- "$patch_file"` and store only the first field: the lowercase 64-character hex digest. The trailing newline printed by `sha256sum` is not part of the digest.
- Verify apply-side digests with the same `sha256sum -b` command and exact string comparison. Do not use `sha256sum -c` against manifest-derived text, because filename parsing would become a second input surface.
- `manifest.json` itself must be a regular file inside the artifact directory, not a symlink, directory, or other non-regular file. `base_commit` and `head_commit` must be full lowercase Git object IDs: 40 hex characters for SHA-1 repositories or 64 hex characters for SHA-256 repositories. Reject refs, abbreviated hashes, rev expressions, and values with control characters.
- Each `patches[].filename` must be a basename ending in `.patch`, with no `/`, no `..`, and no control characters. It must name an existing regular file inside the artifact directory, not a symlink, directory, or other non-regular file. `patch_count` must equal the length of the `patches` array and the number of `.patch` files in the artifact directory.
- Duplicate filenames or non-hex digests are manifest integrity failures and exit 12.

### Exit code summary

```
0   success
1   usage error / SSH failure / general error, including post-apply validation or push failure after commits were applied
10  source worktree dirty
11  source range is not exportable (not ancestor, not ahead, merge commits, or all-empty)
12  manifest/checksum integrity failure
13  repo identity mismatch
14  base commit missing or not ancestor of destination HEAD
15  git am failed
16  destination worktree dirty
```

These start at 10 to avoid collision with `allod change` exit codes (1-6) since they share the `allod` script.

## Agent Gates

- Human reviews the SSH command surface in `fetch` before merging. The remote command must not pass unsanitized user input to a shell.
- Human runs one cross-VM fetch/apply/receive smoke test before merging the tools PR. The automated tests use local SSH mocking, not real VMs, so this blocks merge until the real SSH/tar path has been exercised.

## Acceptance Tests

Test script: `allod/tools/tests/allod-patch.sh`

### CLI routing tests

- `allod --help` lists both `change` and `patch`.
- `allod patch --help` prints patch usage and exits 0.
- `allod patch` with no subcommand exits 1 with patch usage.
- `allod patch nope` exits 1 with an unknown patch command message.

### SSH mock strategy

Tests do not use real SSH. Instead, create a mock `ssh` script on PATH that:
- Parses the host and remote command from args
- Executes the remote command through a shell boundary with stdin/stdout preserved, so quoting and tar-pipe behavior are exercised instead of bypassed by direct helper calls
- Records each remote command string and asserts it is static: each phase's command text is byte-for-byte identical across different source repos, base refs, and remote temp dirs. Raw values and their base64 encodings must not appear in the command text.
- Supports failure modes for connection failure, remote generation failure, remote tar failure, truncated tar output, and cleanup failure

The mock remote filesystem is local test data, but transfer still uses the same stdout tar stream shape as production: mock SSH writes gzipped tar bytes and local `tar -xz` consumes them. This does not cover SSH authentication, login-shell startup files, host key policy, or real network failures; that residual gap is why the Agent Gates require human cross-VM testing before merge.

### fetch tests

- Happy path: single commit ahead of base produces one patch + valid manifest. Verify patch count, filenames, sha256s in manifest, artifact dir exists locally.
- Multiple commits: produces multiple patches in order.
- Target parsing/input validation: empty host, empty source repo, relative source repo, newline-bearing source repo, and newline-bearing `--base` all fail before remote work.
- Dirty source worktree: tracked, staged, and untracked changes each exit 10 with actionable messages.
- No commits to export (HEAD == base): exits 11.
- Source base not ancestor of `HEAD`: exits 11 before `git format-patch`.
- Merge commit in export range: exits 11 before `git format-patch` with a message that non-linear history is unsupported.
- All-empty-commit range: every commit in the range was created with `--allow-empty` (no diff). Exits 11 after `git format-patch` produces zero patches.
- SSH connection failure (mock returns non-zero): exits 1 with message.
- Remote cleanup: after successful fetch, remote temp dir is removed (verify via mock).
- Remote cleanup failure after successful local promotion: exits 1, preserves the local artifact, and prints both local and remote cleanup paths.
- `--output <dir>`: artifacts land in specified directory.
- `--output <dir>` existing path, including a dangling symlink: exits 1 before remote work.
- `--output <dir>` with a missing or unwritable parent directory: exits 1 before remote work.
- Local extraction failure: exits 1, final output dir is absent, local staging is removed, and the remote temp dir path is printed for manual cleanup.
- Local artifact validation failure after extraction: missing, symlinked, or non-regular `manifest.json` exits 1, leaves the final output absent, removes local staging, prints the remote temp dir path, and does not run remote cleanup.
- `--base` custom ref: patches cover the specified range.
- Remote stdout discipline: `git format-patch` filename output does not pollute the fetched temp dir control value.
- Remote temp dir validation: empty, multi-line, relative, and non-`/tmp/allod-patch.XXXXXXXXXX` control values exit 1 before tar or cleanup, leave the final output absent, and never run `rm -rf` against the invalid path.
- Remote command injection guard: source repo paths containing spaces, semicolons, quotes, and `$()` characters are fetched correctly, long source repo paths that force base64 past 76 bytes still transfer as one value, and an invalid `--base` value containing shell metacharacters fails without creating a sentinel file.

### apply tests

- Happy path: clean destination repo, matching remote URL, base commit present as an ancestor of `HEAD`, patches apply cleanly. Verify commits exist after apply.
- Artifact path resolution: run `apply` from a cwd that is neither the artifact directory nor the destination repo. Patches still apply because `git am` receives artifact-rooted paths.
- Checksum mismatch: tamper with a patch file after fetch. Exits 12.
- Manifest validation: malformed JSON, symlinked or non-regular `manifest.json`, missing or wrong-type required fields, duplicate filenames, path traversal filenames, basenames not ending in `.patch`, control-character filenames, non-full or non-hex `base_commit`/`head_commit`, refname or rev-expression commit fields, non-hex digests, missing listed patch files, symlink or non-regular listed patch files, patch count mismatch, and unlisted `.patch` files each exit 12 before `git am`.
- Repo identity mismatch: destination has different origin URL. Exits 13 with both URLs shown.
- Base commit missing: destination is behind. Exits 14 with "git fetch" hint.
- Base commit not ancestor: destination has the base object only on another ref or branch. Exits 14 with checkout/fetch guidance before `git am`.
- Dirty destination worktree: exits 16.
- `git am` conflict: create a conflicting commit in destination before apply. Exits 15; `HEAD` is unchanged, worktree/index are clean, and `.git/rebase-apply`/`.git/rebase-merge` are absent after abort.
- Destination already ahead of base: pre-existing destination commits after `base_commit` do not affect post-apply validation; a pre-existing whitespace error in `base_commit..pre_apply_head` does not fail a clean apply.
- Post-apply whitespace failure: a patch that introduces whitespace errors exits 1 after applying commits, skips push, and prints the repo path plus reset/fix context.
- `--push`: verify git push is called (via mock git wrapper).
- `--push` failure: exits 1, leaves applied commits in place, and prints the repo path plus reset/push context.
- Without `--push`: verify git push is NOT called.
- Multiple patches: all apply in manifest order, log shows correct range.

### receive tests

- Happy path: end-to-end fetch + apply succeeds. Verify commits in destination and the printed receive artifact path exists.
- Artifact handoff: receive passes its explicit `artifact_dir` from fetch to apply without parsing the human-readable fetch summary.
- Fetch failure propagates: dirty source causes exit 10 (not masked), and the receive-owned parent temp dir is removed when no artifact was promoted.
- Fetch cleanup failure after artifact promotion propagates exit 1, preserves and prints the artifact dir, and does not invoke apply.
- Apply failure propagates: checksum mismatch causes exit 12, and the artifact dir remains printed for inspection.
- `--push` passed through to apply.

### Run

```sh
bash allod/tools/tests/allod-patch.sh
```

## Rollback Plan

Revert the commit that adds `patch` namespace to `allod/tools/allod`, the test file, and the docs file. No packaging changes or flake lock updates are in this PR.

If validation already ran the tool:

- Delete local artifact dirs created under `/tmp/allod-patch.*` or any explicit `--output` paths used for testing.
- SSH to the source VM and remove any remote temp dir path printed after a failed transfer.
- If smoke testing applied commits to a disposable destination branch, delete/reset that branch; if it applied to a real branch, use normal git revert/reset workflow chosen by the human.
