## Implementation Plan - Allod Change Workflow

### Goal

A shell script (`allod`) with a `change` command namespace that mechanically enforces the git workflow for codebase changes: worktree isolation, additive commits, safe pushes, and PR handoff. The initial action set is `allod change begin`, `allod change ship`, and `allod change cleanup`, callable by any LLM agent harness or human.

### Scope

**In scope:**
- `allod/tools/allod` — new bash script (~300 lines)
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
- If protected: `git worktree add <tmpdir> -b agent/<desc>` from default branch, prints worktree path
- If not protected: prints repo path (no-op — no worktree, no branch)
- Exits non-zero with actionable message if `-d` is missing for a protected repo

The caller captures the printed path and operates there.

Worktree path: `/tmp/allod-change-<repo>-<desc>-XXXXXX` (via mktemp). Human-readable prefix for debugging, random suffix for uniqueness.

#### `allod change ship [options]`

Runs from inside the working directory (worktree or repo). Stages, commits, pushes, optionally creates PR.

```
Options:
  -m, --message <text>       Commit message
  -M, --message-file <file>  Commit message from file (- for stdin)
  -f, --files <file,...>     Files to stage (default: git add -u, tracked modified files)
  -t, --title <text>         PR title (default: first line of commit message)
  -b, --body <text>          PR body text
  -B, --body-file <file>     PR body from file (- for stdin)
  --base <branch>            PR base branch (default: repo default branch)
  --depends-on <text>        Appends "Depends on: <text>" line to PR body
  --commit-only              Force commit+push only, skip PR even if body provided
  --dry-run                  Print plan, don't execute
```

Behavior:
1. Stage files (`-f` list, or `git add -u` for all tracked modified)
2. Refuse if nothing staged (exit 4, actionable message)
3. `git commit` with message (never `--amend`)
4. `git push -u origin HEAD` (never `--force`, never `--force-with-lease`)
5. If body provided and not `--commit-only`:
   - Validate body contains `## Validation` heading (exit 3 if missing)
   - If `--depends-on`, append dependency line
   - `forge pr create -t <title> -H <branch> [-B <base>] -F <body-file>`
   - Print PR URL
6. Print summary: branch, commit SHA, PR URL (if created)

Exit codes:
```
0  success
1  usage error / missing required args
2  would commit to protected branch directly (use allod change begin first)
3  PR body missing ## Validation section
4  nothing to commit
```

Safety check: if cwd is on a protected branch's default branch (e.g. master on allod/tools), `ship` refuses with exit 2 and tells the user to run `allod change begin` first. This catches the case where the agent forgot to isolate.

#### `allod change cleanup <worktree-path>`

Removes a worktree after the PR is merged.

- `git worktree remove <path>` if clean
- If dirty: warns and exits non-zero (user decides whether to force)
- Runs `git worktree prune` in the parent repo

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
- Protected repo → creates worktree + agent branch, prints path
- Protected repo, missing `-d` → exits non-zero with message
- Non-protected repo → prints repo path, no worktree created
- Called twice with same description → second call gets unique worktree (mktemp suffix)

**ship tests:**
- Non-protected: stages, commits, pushes to current branch
- Protected worktree: stages, commits, pushes agent/* branch
- On protected default branch without begin → exits 2
- With PR body containing ## Validation → calls forge pr create with correct args
- PR body missing ## Validation → exits 3
- Nothing to commit → exits 4
- `--commit-only` with body → commits and pushes, no forge call
- `--depends-on` → appended to body passed to forge
- `--dry-run` → prints plan, no git state changes
- `--files` → stages only specified files
- Never amends (verify by checking commit count)
- Never force-pushes (verify push args passed to git)

**cleanup tests:**
- Clean worktree → removed successfully
- Dirty worktree → warns, exits non-zero

**Smoke test (manual):**
```bash
# From any repo:
path=$(allod change begin -d "test" /home/vnprc/work/allod/tools)
cd "$path"
echo "test" > scratch.txt
git add scratch.txt
allod change ship -m "test commit" --commit-only
git log --oneline -1  # shows "test commit"
allod change cleanup "$path"
```

### Rollback Plan

New script and test file in allod/tools. `git revert` the commit that adds them. No existing workflow files modified, no config changes, no side effects. Worktrees created by the tool are in /tmp and self-clean on reboot.

### Naming and Action Follow-up

Before implementation, revisit the action vocabulary. `allod change ship` keeps the first version compact by combining stage, commit, push, and optional PR creation, but the workflow may be clearer if `ship` is split into separate actions such as `allod change record` for additive commit+push and `allod change submit` for PR creation/review handoff. The command name should keep loading the right context for both humans and agents: an Allod-managed codebase change, not an official Git subcommand and not an agent-only tool.
