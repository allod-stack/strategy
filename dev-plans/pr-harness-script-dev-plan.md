## Implementation Plan - Allod Change Workflow

### Goal

A shell script (`allod`) with a `change` command namespace that mechanically enforces the git workflow for codebase changes: worktree isolation, additive commits, safe pushes, and PR handoff. The action set is `allod change begin`, `allod change record`, `allod change submit`, and `allod change cleanup`, callable by any LLM agent harness or human.

### Scope

**In scope:**
- `allod/tools/allod` — new bash script
- `allod/tools/tests/allod-change.sh` — tests

**Out of scope:**
- Multi-repo sequencing (caller invokes once per repo, handles ordering)
- PR commenting (caller uses `forge pr comment` directly)
- Plan review orchestration (that's the agent's job)
- Pre-push hook (issue #35, separate concern)

### Interface Contracts

#### `allod change begin [-d <description>] [<repo-path>]`

Prepares a workspace. Checks protected-branches and sets up isolation if needed.

- Resolves repo path (arg or cwd), validates it's a git repo
- Derives the repo's `$HOME`-relative path for protected-branches lookup: from a worktree, use `git rev-parse --path-format=absolute --git-common-dir` to find the main repo's `.git` dir, take its parent; from a regular repo, use `--show-toplevel`. If the resolved repo path is under `$HOME`, strip `$HOME/` to get the lookup key (e.g., `work/allod/tools`). If it is outside `$HOME`, it does not match the protected-branches file and is treated as unprotected.
- Reads `~/.config/git/protected-branches`, matches the `$HOME`-relative path
- If not protected: prints repo path (no-op — no fetch, no worktree, no branch)
- If protected: fetches from origin, then `git worktree add <tmpdir> -b agent/<desc> origin/<protected-branch>` where `<protected-branch>` is the branch named in the protected-branches file entry (not from `git symbolic-ref refs/remotes/origin/HEAD`). Prints worktree path.
- Exits non-zero with actionable message if `-d` is missing for a protected repo
- Validates the `-d` value in two layers: it must match `[a-zA-Z0-9._-]+` (no spaces or slashes), and the resulting `agent/<desc>` branch must pass `git check-ref-format --branch`. Exits 1 with actionable message if invalid.
- Exits non-zero with actionable message if `agent/<desc>` branch already exists

The caller captures the printed path and operates there.

Worktree path: `/tmp/allod-change-<repo-slug>-<desc>-XXXXXX` (via mktemp). `<repo-slug>` is derived from the protected-branches lookup key by replacing every character outside `[a-zA-Z0-9._-]` with `-` (for example, `work/allod/tools` becomes `work-allod-tools`). Repos outside `$HOME` use the resolved repo basename with the same sanitization. The mktemp template must never contain slashes from the repo key. Human-readable prefix for debugging, random suffix for uniqueness.

#### `allod change record [options]`

Runs from inside the working directory (worktree or repo). Stages, commits, and pushes. Never creates a PR.

```
Options:
  -m, --message <text>       Commit message (required; must be non-empty)
  -M, --message-file <file>  Commit message from file (- for stdin)
  -f, --files <file>         Files to stage (repeatable; default: git add -u)
```

Behavior:
0. Before staging, resolve the current branch. If HEAD is detached, exit 1 with an actionable message. Also run the protected-branch safety check before staging or committing so a refused command leaves the index untouched.
1. Stage files (`-f` list, or `git add -u` for all tracked modified)
2. If nothing staged: check for unpushed commits. Use `git log @{u}..HEAD` when an upstream exists; if no upstream exists yet, handle that explicitly instead of failing on `@{u}` (first-push retry after a failed `git push -u origin HEAD` is the important case). If unpushed commits exist, skip to step 5 (retry push). Otherwise exit 4 with actionable message.
3. Refuse if commit message is empty (exit 1)
4. `git commit` with message (never `--amend`)
5. `git push -u origin HEAD` (never `--force`, never `--force-with-lease`)
6. Print summary: branch, commit SHA

Protected-branch safety check: resolve the repo's `$HOME`-relative path (same logic as `begin` — use `--git-common-dir` from worktrees). If the current branch matches the protected branch for this repo in `~/.config/git/protected-branches`, refuse with exit 2 and tell the user to run `allod change begin` first.

#### `allod change submit [options]`

Runs from inside the working directory (worktree or repo). Creates a PR for the current branch. Does not commit or push — the caller must `record` first.

```
Options:
  -t, --title <text>         PR title (required)
  -b, --body <text>          PR body text
  -F, --body-file <file>     PR body from file (- for stdin)
  --base <branch>            PR base branch (default: repo default branch)
  --depends-on <text>        Appends "Depends on: <text>" line to PR body
  --dry-run                  Print plan, don't execute
```

Behavior:
1. Resolve the current branch. If HEAD is detached, exit 1 with an actionable message.
2. Fail early if `forge` is not on PATH (exit 1)
3. Validate body contains `## Validation` heading (exit 3 if missing)
4. If `--depends-on`, append dependency line to body
5. Check for an existing PR: `forge pr find-by-head <branch>` returns the PR number if an open PR exists, empty otherwise. If found, exit 6 with message directing the agent to use `forge pr edit` instead
6. `forge pr create -t <title> -H <branch> [-B <base>] -F <body-file>`
7. Print PR URL

`--dry-run` behavior: runs steps 1, 3, and 4 (branch resolution, body validation, depends-on assembly) but skips steps 2, 5, and 6 (forge availability check, existing-PR check, PR creation). Prints the resolved branch, base, title, and assembled body. Does not require `forge` on PATH.

#### `allod change cleanup <worktree-path>`

Removes a worktree after the PR is merged.

- If dirty: warns and exits non-zero (user decides whether to force)
- If the worktree has unpushed commits: warns and exits non-zero (same upstream/no-upstream detection as `record`)
- Records the current branch, then `git worktree remove <path>` if clean
- Deletes the local `agent/*` branch with `git branch -D <branch>` after removing the worktree; the unpushed-commit check is the safety gate. Does not delete the remote branch.
- Runs `git worktree prune` in the parent repo

Exit codes (shared across all subcommands):
```
0  success
1  usage error / missing required args / empty message / detached HEAD / forge not found / cleanup refused
2  would commit to protected branch directly (use allod change begin first)
3  PR body missing ## Validation section
4  nothing to commit
5  branch already exists (use a different -d or clean up)
6  PR already exists for this branch (use forge pr edit)
```

### Reusable code

- `allod/tools/lib/workspace.sh` — `workspace_is_repo_root()` for repo validation
- `allod/tools/forge` — called as an external command for PR creation, not sourced
- `~/.config/git/protected-branches` — existing config file, no format changes needed

### Agent Gates

None. The script and its tests run entirely in the dev VM.

### Acceptance Tests

Test script: `allod/tools/tests/allod-change.sh`

Setup: temporary git repos with a local "remote" (bare repo), mock `protected-branches` file, mock `forge` that records its arguments.

**begin tests:**
- Protected repo → fetches, creates worktree + agent branch from the protected-branches entry's `origin/<branch>`, prints path
- Protected repo, missing `-d` → exits non-zero with message
- Non-protected repo → prints repo path, no worktree created
- Non-protected repo outside `$HOME` with no origin → prints repo path, no fetch, no worktree created
- Called twice with same description → second call fails with exit 5 and actionable message
- Branch already exists on remote → fails with exit 5
- Protected repo with a nested `$HOME`-relative path such as `work/allod/tools` → creates the worktree under `/tmp` using a sanitized repo slug, not nested directories
- Invalid `-d` value (spaces, slashes, empty, or git-invalid values such as `.foo`, `foo..bar`, `foo.lock`) → exits 1 with actionable message before any git operations

**record tests:**
- Non-protected: stages, commits, pushes to current branch
- Protected worktree: stages, commits, pushes agent/* branch
- On protected default branch without begin → exits 2 before staging and leaves the index unchanged
- Detached HEAD → exits 1 with actionable message
- Nothing to commit → exits 4
- Empty commit message → exits 1
- `-f file1 -f file2` → stages only specified files
- Without `-f` → stages all tracked modified (`git add -u`)
- After a failed push leaves an unpushed local commit, rerunning `record` with no staged changes retries the push
- With existing unpushed commits and new staged changes, `record` creates a new commit and pushes both commits
- On a branch with no upstream yet, the unpushed-commit check does not fail on `@{u}` and the push retry succeeds
- Never amends (verify by checking commit count)
- Never force-pushes (verify push args passed to git)

**submit tests:**
- With body containing ## Validation → calls forge pr create with correct args
- Body missing ## Validation → exits 3
- `--depends-on` → appended to body passed to forge
- `--dry-run` → prints resolved branch, base, title, and assembled body; no forge call; succeeds even when forge is not on PATH
- `--dry-run` with body missing ## Validation → exits 3 (local validations still run)
- forge not on PATH → exits 1 with message
- PR already exists for branch → exits 6 with message directing to forge pr edit
- Not on a branch (detached HEAD) → exits with actionable message

**cleanup tests:**
- Clean worktree → worktree removed successfully and local agent branch deleted
- Dirty worktree → warns, exits non-zero
- Clean worktree with unpushed commits → warns, exits non-zero

**Smoke test (manual):**
```bash
# From any repo:
desc="smoke-$(date +%s)"
path=$(allod change begin -d "$desc" /home/vnprc/work/allod/tools)
cd "$path"
echo "test" > scratch.txt
allod change record -m "test commit" -f scratch.txt
branch=$(git branch --show-current)
git log --oneline -1  # shows "test commit"
cd -
allod change cleanup "$path"
git -C /home/vnprc/work/allod/tools push origin ":$branch"  # remove the smoke-test branch
```

### Rollback Plan

New script and test file in allod/tools. `git revert` the commit that adds them. No existing workflow files modified, no config changes, no side effects. Worktrees created by the tool are in /tmp and self-clean on reboot.
