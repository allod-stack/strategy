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
- Reads `~/.config/git/protected-branches`, matches repo's `~/work/`-relative path
- Fetches from origin before creating the worktree so the base is current
- If protected: `git worktree add <tmpdir> -b agent/<desc>` from `origin/<default-branch>`, prints worktree path
- If not protected: prints repo path (no-op — no worktree, no branch)
- Exits non-zero with actionable message if `-d` is missing for a protected repo
- Exits non-zero with actionable message if `agent/<desc>` branch already exists

The caller captures the printed path and operates there.

Worktree path: `/tmp/allod-change-<repo>-<desc>-XXXXXX` (via mktemp). Human-readable prefix for debugging, random suffix for uniqueness.

#### `allod change record [options]`

Runs from inside the working directory (worktree or repo). Stages, commits, and pushes. Never creates a PR.

```
Options:
  -m, --message <text>       Commit message (required; must be non-empty)
  -M, --message-file <file>  Commit message from file (- for stdin)
  -f, --files <file>         Files to stage (repeatable; default: git add -u)
```

Behavior:
1. Stage files (`-f` list, or `git add -u` for all tracked modified)
2. Refuse if nothing staged (exit 4, actionable message)
3. Refuse if commit message is empty (exit 1)
4. `git commit` with message (never `--amend`)
5. `git push -u origin HEAD` (never `--force`, never `--force-with-lease`)
6. Print summary: branch, commit SHA

Safety check: if cwd is on a protected branch's default branch (e.g. master on allod/tools), `record` refuses with exit 2 and tells the user to run `allod change begin` first.

#### `allod change submit [options]`

Runs from inside the working directory (worktree or repo). Creates a PR for the current branch. Does not commit or push — the caller must `record` first.

```
Options:
  -t, --title <text>         PR title (required)
  -b, --body <text>          PR body text
  -B, --body-file <file>     PR body from file (- for stdin)
  --base <branch>            PR base branch (default: repo default branch)
  --depends-on <text>        Appends "Depends on: <text>" line to PR body
  --dry-run                  Print plan, don't execute
```

Behavior:
1. Fail early if `forge` is not on PATH (exit 1)
2. Validate body contains `## Validation` heading (exit 3 if missing)
3. If `--depends-on`, append dependency line to body
4. Check whether a PR already exists for the current branch; if so, exit with message directing the agent to use `forge pr edit` instead
5. `forge pr create -t <title> -H <branch> [-B <base>] -F <body-file>`
6. Print PR URL

#### `allod change cleanup <worktree-path>`

Removes a worktree after the PR is merged.

- `git worktree remove <path>` if clean
- If dirty: warns and exits non-zero (user decides whether to force)
- Runs `git worktree prune` in the parent repo

Exit codes (shared across all subcommands):
```
0  success
1  usage error / missing required args / empty message / forge not found
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
- Protected repo → fetches, creates worktree + agent branch from origin default, prints path
- Protected repo, missing `-d` → exits non-zero with message
- Non-protected repo → prints repo path, no worktree created
- Called twice with same description → second call fails with exit 5 and actionable message
- Branch already exists on remote → fails with exit 5

**record tests:**
- Non-protected: stages, commits, pushes to current branch
- Protected worktree: stages, commits, pushes agent/* branch
- On protected default branch without begin → exits 2
- Nothing to commit → exits 4
- Empty commit message → exits 1
- `-f file1 -f file2` → stages only specified files
- Without `-f` → stages all tracked modified (`git add -u`)
- Never amends (verify by checking commit count)
- Never force-pushes (verify push args passed to git)

**submit tests:**
- With body containing ## Validation → calls forge pr create with correct args
- Body missing ## Validation → exits 3
- `--depends-on` → appended to body passed to forge
- `--dry-run` → prints plan, no forge call
- forge not on PATH → exits 1 with message
- PR already exists for branch → exits 6 with message directing to forge pr edit
- Not on a branch (detached HEAD) → exits with actionable message

**cleanup tests:**
- Clean worktree → removed successfully
- Dirty worktree → warns, exits non-zero

**Smoke test (manual):**
```bash
# From any repo:
path=$(allod change begin -d "test" /home/vnprc/work/allod/tools)
cd "$path"
echo "test" > scratch.txt
allod change record -m "test commit" -f scratch.txt
git log --oneline -1  # shows "test commit"
allod change cleanup "$path"
```

### Rollback Plan

New script and test file in allod/tools. `git revert` the commit that adds them. No existing workflow files modified, no config changes, no side effects. Worktrees created by the tool are in /tmp and self-clean on reboot.
