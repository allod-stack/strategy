# allod change begin: Unconditional Worktree Isolation

## Tracking Issue

https://forge.anarch.diy/allod/tools/issues/116

One PR in `allod/tools` carrying `Closes allod/tools#116` and
`Refs allod/tools#115`. A separate `allod/memory` commit follows the merge and
carries no closing keyword; it is not part of the tools PR.

This plan is a child of `dev-plans/concurrent-agent-workspace.md` and inherits
contracts C1 (worktree siting), C2 (worktree detection), C3 (branch handoff),
C4 (`-d` as the isolation switch), and C5 (frozen collector output). They are
not restated here. This plan owns the implementation detail beneath them and
settles the one question C4 left open.

## Goal

`allod change begin -d <description>` gives every repo its own worktree and
`agent/<description>` branch regardless of branch protection, and every worktree
it has ever created can be listed and reclaimed without hunting the filesystem
for it.

## Scope

In scope — one `allod/tools` PR:

- `change_begin` (`allod:240-301`): drop the unprotected passthrough
  (`allod:268-272`), key isolation on `-d`, resolve the worktree base branch for
  repos with no `protected-branches` entry, site the worktree per C1, and write
  the C3 handoff file.
- A new `change_list` plus its dispatch case (`allod:523-530`) and usage entry
  (`allod:31-40`): the orphan enumeration required by the issue's fourth goal.
- `tests/allod-change.sh`: the assertions named below, plus new positive and
  negative coverage for the flipped path and for `list`.
- `README.md`: a "Making a change" block under Workflow (lines 40-77) stating
  the `-d` contract, the `~/changes` siting, and the reclaim path.

Out of scope:

- `change_cleanup` (`allod:482-513`). Its refusals and its removal semantics are
  unchanged; `list` is built to agree with it rather than to replace it.
- `lib/workspace.sh` collector changes, which are allod/tools#117 under C5.
- `record`'s reader half of C3 and its concurrency guards, allod/tools#118.
- Hook-level enforcement, allod/tools#119 and allod/tools#112.
- Moving or migrating the two `/tmp/allod-change-*` worktrees that exist on the
  VM today. They stay where they are; `list` finds them because it derives from
  git rather than from a path, and `cleanup` already reclaims them.
- The dead `|| echo "master"` fallback in `workspace_repo_default_branch`
  (`lib/workspace.sh:33-38`), which never fires — see the corrections below.
  `begin` defends against its empty return instead of changing a helper shared
  with `pull-all`, `flake-status`, and `flake-update-cascade`.

Separate `allod/memory` commit, after the tools PR merges and after the rebuild
gate below: `memory.md:49`, `git-workflow.md:9`, `git-workflow.md:13`,
`git-workflow.md:28`, `allod.md:62`, and a `list` entry beside `allod.md:65`.

## Risk Assessment

Residual risk: R2 Medium. This agrees with the arc triage and with the review
depth it sets — dev plan plus one review pass by a different model.

Why:

- The blast radius is one CLI in one repo, and within it one function plus one
  new read-only function. Nothing here touches hooks, provisioning, secrets,
  generated lifecycle artifacts, or forge state.
- The newly default action is additive: `git worktree add` creates a directory
  and a branch and never moves the shared checkout's HEAD, index, or working
  tree. That is the property that makes the flip safe to run while another agent
  is mid-write in the same repo, and it is asserted directly.
- `begin` reaches the network only through `ls-remote` and `fetch`, both reads.
  It never pushes; publication risk is unchanged.
- The one destructive command in the namespace, `cleanup`, is untouched, and the
  new `list` is strictly read-only. An orphan is never removed implicitly.
- Rollback is a straight revert with no persistent state to unwind, since a
  reverted `begin` still returns a usable path and existing worktrees remain
  valid (see Rollback Plan).

Why not R1: `begin` is the entry point of every change workflow on the VM, and
the change alters what a documented command does for repos that previously got a
plain path back. The correctness of the flip depends on doc and memory updates
landing in the right order across two repos, which is cross-repo sequencing.

Why not R3: no security or privacy boundary moves, no cross-repo interface is
introduced, the branch-protection rails are untouched, and every path including
the failure paths is exercisable in the existing fixture harness.

Human scrutiny, in order:

- The `-d` absent branch of `change_begin`. Confirm the in-place flow for
  `allod/memory` and `allod/strategy` is byte-for-byte what it was: a printed
  path, no worktree, no branch, no fetch.
- The failure handling around `git worktree add`. Confirm the plan's refusal to
  delete a branch it cannot prove it created (below) reads as deliberate.
- The `list` state words against `cleanup`'s refusals, since the contract is
  that they agree.
- The `allod/memory` commit, which must not land before the rebuild.

### Verified corrections to the issue body and the arc contracts

Each of these was checked against the code or the running VM. None of them
changes C1–C5; two change what this plan has to implement.

**`/tmp` is swept, not merely uncleaned.** The issue says `/tmp` has "no
tmpfiles cleanup rule" and that orphans "accumulate indefinitely". The VM has
`/etc/tmpfiles.d/tmp.conf` with `q /tmp 1777 root root 10d`, and
`systemd-tmpfiles-clean.timer` is active. So `/tmp` worktrees idle for ten days
are deleted, which is worse than accumulation: it silently destroys a stalled
agent's uncommitted work and leaves a `prunable` stub in
`<repo>/.git/worktrees/`. The conclusion is unchanged and C1 stands, but the
reason is now "a hidden ten-day deadline on unique local work" rather than
"clutter", and it makes the orphan states below a real classification problem
rather than a cosmetic one.

**The two named assertions do not flip; they get renamed, and a third one
does.** `tests/allod-change.sh:256` and `:262` are reached by
`allod change begin <repo>` with no `-d` (`:254`, `:260`). Under C4 that is
still a passthrough, so both assertions keep passing unchanged — what is stale
is their descriptions, which say "non-protected repo" and "treated as
unprotected" for behavior that is now keyed on `-d`. The assertion that actually
has to move is `tests/allod-change.sh:287`, which pins
`/tmp/allod-change-work-allod-tools-<desc>-*` and must become
`$HOME/changes/work-allod-tools-<desc>-*` under C1. The harness `/tmp` sweep at
`tests/allod-change.sh:20` becomes dead once worktrees land under `$HOME`, which
the harness already sets to a temp dir it removes. What the old contract really
rests on is the *absence* of any test for `begin -d` on an unprotected repo;
that gap is the main new coverage.

**`allod/tools` has no doc that says protected repos get worktrees.** The only
in-repo text describing `begin` is `change_usage` (`allod:31-40`), which names
`-d` as optional and says nothing about protection; `README.md` has no
`allod change` section, and `docs/` holds only `allod-patch.md` and `forge.md`.
So the tools-side doc work is additive, not corrective. The corrective work is
all in `allod/memory`, and there are five sites there, not two: `memory.md:49`,
`git-workflow.md:9`, `git-workflow.md:13`, `git-workflow.md:28`, and
`allod.md:62`.

**`workspace_repo_default_branch` returns empty, not `master`, when
`origin/HEAD` is unset.** Its `|| echo "master"` binds to the pipeline, whose
exit status is `sed`'s and therefore always zero, so the fallback is
unreachable. Today `begin` never calls it — the base comes from the
`protected-branches` entry. After the flip, every unprotected repo's base comes
from this helper, so `begin` inherits a landmine that produces
`git worktree add ... origin/` and a confusing git error. `begin` must validate
the resolved base and fail with its own message.

**`begin -d` now depends on `origin` existing.** The fixture at
`tests/allod-change.sh:258-262` uses a repo with no remote. With no `-d` it
still passes through, but `begin -d` on such a repo would reach
`ref_exists_remote` (`allod:170-186`) and die with "could not check origin for
branch", which does not name the actual problem. A preflight is needed.

**Scale.** The arc quotes "12 of 19 checkouts" as unprotected. This VM has 11
registry checkouts and 8 protected entries, so the flip changes `begin -d`
behavior for three repos: `allod/memory`, `allod/strategy`, and
`allod/.profile`. That lowers the immediate blast radius and it also means the
fixtures, not the live workspace, are where this gets exercised.

## Interface Contracts

Inherited from `dev-plans/concurrent-agent-workspace.md`: C1, C2, C3, C4, C5.
C1 and C3 are implemented by this PR; C2 is consumed by it; C4 is completed by
it; C5 is not touched.

### `allod change begin [-d <description>] [<repo-path>]`

Option parsing (`allod:244-265`) is unchanged. After
`repo=$(resolve_git_repo ...)` and
`protected_branch=$(protected_branch_for_dir "$repo" || true)`:

With `-d` absent:

- If `protected_branch` is non-empty, exit 1 with a message that still contains
  the substring `-d`. See the settlement below.
- Otherwise print `$repo` and return 0. No fetch, no branch, no worktree, no
  handoff file, no directory created anywhere. This is the entire in-place
  default-branch flow and it must not acquire a single new side effect.

With `-d` present, for every repo, protected or not:

1. `validate_description` and `branch="agent/$desc"`, unchanged
   (`allod:276-277`).
2. Preflight `git -C "$repo" remote get-url origin`. On failure exit 1 naming
   the repo and stating that an isolated change needs an `origin` remote.
3. `base="${protected_branch:-$(default_remote_branch_for_dir "$repo")}"`. A
   `protected-branches` entry still names the base for the repos that have one,
   so protected-repo behavior is bit-identical to today. If `base` is empty exit
   1 naming `git -C <repo> remote set-head origin -a` as the repair.
4. `ref_exists_local` then `ref_exists_remote`, both exit 5, unchanged
   (`allod:279-284`). These are the duplicate-description rails and they now
   cover every repo rather than only protected ones.
5. `git -C "$repo" fetch origin`, unchanged (`allod:286-290`).
6. After the fetch, verify `refs/remotes/origin/$base` resolves. If not, exit 1
   naming the missing ref.
7. `mkdir -p "$CHANGES_DIR"` then
   `worktree_path=$(mktemp -d "$CHANGES_DIR/${slug}-${desc}-XXXXXX")`, where
   `CHANGES_DIR="${HOME}/changes"` is a constant beside
   `PROTECTED_BRANCHES_FILE` (`allod:9`). No environment override: C1 forbids
   consumers depending on the path, and an override would be a second source of
   truth for it. `slug` comes from `repo_slug_for_dir` unchanged
   (`allod:145-154`), and the `allod-change-` prefix in the old name is dropped
   per C1.
8. `git -C "$repo" worktree add "$worktree_path" -b "$branch" "origin/$base"`.
9. C3: write `$branch` to
   `$(git rev-parse --path-format=absolute --git-dir)/allod-change-branch`
   resolved inside the new worktree, which is its private
   `<main>/.git/worktrees/<name>` directory.
   A failed write is fatal and rolls the worktree back, because a missing file
   means "begin was never called" to allod/tools#118's reader and would
   downgrade a guard silently.
10. Print `$worktree_path`. Unchanged; the stdout contract that
    `path=$(allod change begin ...)` depends on is preserved exactly.

Failure handling for step 8 or 9: `rm -rf "$worktree_path"` and
`git -C "$repo" worktree prune`, then exit with git's status. Both are provably
safe — prune only drops admin entries whose gitdir is gone. The branch is *not*
deleted. The pre-check at step 4 proves the branch did not exist before this
call, but under concurrency another agent may have created it between the check
and the add, and deleting it would be the exact failure class allod/tools#115
exists to close. Instead the error states that `agent/<description>` may have
been left behind and points at `allod change list`. This is architecture
principle 11: when state is left uncertain, say so loudly rather than guess.

### The C4 open question, settled

**Omitting `-d` on a protected repo still fails loud, and the check stays keyed
on the repo rather than on HEAD.** The refusal at `allod:274-275` survives with
a reworded message; the exit code stays 1.

The arc's argument holds and the code confirms it.
`refuse_protected_branch_record` (`allod:230-238`) dies 2 when the current
branch equals the repo's protected branch, so if `begin` handed back the path
the very next command in the documented flow is guaranteed to fail. Failing at
`begin` is earlier, and it fails on the operator's actual intent — "give me a
path to commit in place on" — rather than three steps later on a symptom.
Deleting the refusal would replace a loud early error with a silent success that
strands the caller, which is a straight principle 11 regression.

The counter-argument is that C4 states `-d` absent means "print the shared
checkout path and create nothing", with no exception, and that a protected-repo
exception leaves `protected-branches` steering `begin`'s control flow. The
distinction that resolves it: C4 removes protection as the *isolation* switch,
not as a *validation* input. `begin` without `-d` is a request to commit in
place on the current branch, and validating that request against the protection
list is exactly the job C4 leaves the file — "governing only which branch is
protected". The check and `record`'s check are then the same rule applied at two
points, which is defence in depth, not leftover coupling.

Keeping it keyed on the repo rather than on the shared checkout's current branch
is the second half of the settlement. A HEAD-keyed check would be a closer
mirror of `record` and would stop refusing when the shared checkout happens to
sit on a feature branch — but under this arc's premise that state is itself the
bug: an in-place commit is legitimate only on the repo's default branch, so if
that branch is protected, no in-place change in that checkout is legitimate
regardless of where HEAD points. A repo-keyed check is also deterministic, while
a HEAD-keyed one makes `begin`'s outcome depend on mutable shared state that
another agent can move between the check and its use.

### `allod change list [<repo-path>]` and orphan reclaim

New read-only subcommand, dispatched from `change_main` (`allod:523-530`).

Enumeration. With no argument, iterate `workspace_collect_repos "$WORK_DIR"` —
already available, since `allod` sources `lib/workspace.sh` at `allod:4-7` — and
for each registry checkout run `git -C <repo> worktree list --porcelain`,
skipping the first record, which is always the main checkout (C2, verified from
inside a worktree as well as from the main one). With a `<repo-path>` argument,
resolve it with `resolve_git_repo` and enumerate only that repo. Never glob a
directory: that is what makes `list` find the `/tmp` worktrees that predate C1
and anything sited elsewhere later.

If allod/tools#117 has already landed its per-repo worktree helper in
`lib/workspace.sh`, `list` calls it. If this PR lands first, `list` inlines the
ten-line enumeration and allod/tools#117's PR extracts it and repoints `list` at
the helper. Either way exactly one implementation exists once both have landed.

Output. One line per linked worktree: repo name, worktree path, branch (or
`(detached)`), and one state word. Exit 0 whenever enumeration succeeds,
including when there are no worktrees at all — this is a report, and a report
that exits non-zero on findings breaks every `set -e` caller.

States, computed from local git only. No forge call: a tool that tells you what
you may delete must not need the network to answer.

| State | Condition | Reclaim |
|---|---|---|
| `clean` | `git status --porcelain` empty and `has_unpushed_commits` false | `allod change cleanup <path>` |
| `dirty` | `git status --porcelain` non-empty | none automatic |
| `unpushed` | `has_unpushed_commits` true (`allod:203-228`) | none automatic |
| `prunable` | `prunable` annotation in the porcelain output | `git -C <repo> worktree prune` |
| `locked` | `locked` annotation | none; prune skips locked worktrees |

The binding contract: **`list` reports `clean` if and only if
`allod change cleanup` on that path would succeed.** Both read the same two
conditions, which are `cleanup`'s two refusals (`allod:497-499`,
`allod:501-503`), and the acceptance tests assert the agreement directly for
each state rather than asserting each side separately. That is what keeps the
report from drifting away from the command it recommends.

What happens to a dirty or unpushed orphan: nothing automatic, ever. `list`
names it, names why, and names the command that would resolve it; `cleanup`
keeps refusing it. There is no `--force`, no `--all`, and no age or activity
heuristic, for a reason that is observable on this VM right now: of the two
`/tmp` worktrees currently registered against `allod/tools`, one is three days
old and one is minutes old and belongs to a running agent, and no local signal
distinguishes "the agent died" from "the agent is mid-write". Any automatic
reclaim would therefore be a guess that can destroy unique unreconstructable
work, and a bypass flag would be exactly the rail-override that architecture
principle 4 says tools do not ship. The issue asks that orphans be *reclaimable*
without hunting; hunting is the enumeration problem, and `list` is the whole
answer to it. Reclaim stays one explicit `cleanup <path>` per worktree.

A `prunable` stub is the residue of the ten-day `/tmp` sweep described above:
the directory is gone, the admin entry remains, and `cleanup` cannot even be
pointed at it because `resolve_git_repo` requires an existing directory
(`allod:75`). `list` reports it and names `git worktree prune`; `cleanup` also
prunes as its last step (`allod:512`), so any successful reclaim in that repo
clears the stubs as a side effect. No new `allod` surface for it.

Considered and rejected: folding the listing into `cleanup` with no argument.
`cleanup` today prints usage and exits 1 in that case, and turning a destructive
command's zero-argument form into a report is a surprise in the wrong direction.
A separate verb also keeps `list` provably read-only, which is worth more at
review time than one fewer subcommand.

Relationship to allod/tools#117: `work-diff` will show worktree *contents* —
what changed. `list` shows worktree *lifecycle* — whether it can be reclaimed
and how. Different questions, different consumers, no shared output.

### Unchanged

`change_record`, `change_submit`, `change_cleanup`,
`refuse_protected_branch_record`,
`protected_branch_for_dir`, `repo_lookup_key_for_dir`, `main_repo_dir_for_dir`,
`repo_slug_for_dir`, the `protected-branches` file format, and every existing
exit code: 1 usage or validation, 2 protected-branch refusal, 4 nothing to
commit, 5 branch already exists, 6 PR already exists.

## Agent Gates

- **Nothing here needs a rebuild to validate.** Unlike the hook issues in this
  arc, `allod` is validated by invoking the checkout directly, which is what
  `tests/allod-change.sh:5` already does. The agent can run the full acceptance
  set with no host involvement.
- **A rebuild is needed for the change to take effect.** `allod` on the VM
  resolves to `/etc/profiles/per-user/allod/bin/allod`, a nix store path, so
  merging the PR changes nothing for running agents until the human rebuilds.
  This orders the `allod/memory` commit: memory records the current state of the
  world, so the doc update lands after the rebuild, not after the merge.
  Otherwise agents follow instructions the installed tool does not implement.
  Blocks: the memory commit only.
- **The `allod/memory` commit is a public-repo write.** If the implementing
  agent holds private material it cannot push it, and the relay in
  `agent-behavior.md` applies: commit on an `agent/<description>` branch in the
  public checkout, leave it unpushed and clean, and hand the human one
  `allod patch receive` command.
- **Do not run `begin -d` against the live workspace to validate.** It creates a
  real branch and a real worktree in a repo another agent may be using. Every
  path in this plan is exercisable in `tests/allod-change.sh`, which builds its
  own repos under a temporary `HOME`.
- No forge-side branch protection change, no secrets, no provisioning. The
  arc-level gates on those remain with allod/tools#112 and allod/tools#119.

## Acceptance Tests

```sh
cd /home/allod/work/allod/tools
bash -n allod tests/allod-change.sh
bash tests/allod-change.sh
./allod change --help
./allod change list          # read-only against the live workspace
```

`allod change list` is safe to run against the live workspace because it only
reads; it should show the two `/tmp/allod-change-*` worktrees registered against
`allod/tools` today, which is the direct demonstration that enumeration is
git-derived rather than path-derived.

Required fixture coverage in `tests/allod-change.sh`. Positive paths:

- `begin -d` on an **unprotected** repo creates a worktree under
  `$HOME/changes/<slug>-<desc>-XXXXXX`, checked out on `agent/<desc>`, with
  `origin/<default>` an ancestor of HEAD. This is the flip; there is no such
  test today.
- `begin -d` on a **protected** repo is unchanged: the existing assertions at
  `tests/allod-change.sh:239-246` still pass, and `:287` now matches
  `$HOME/changes/work-allod-tools-<desc>-*`.
- C3: the new worktree contains `allod-change-branch` inside its private git
  dir, its content is `agent/<desc>`, and `--git-dir` differs from
  `--git-common-dir` at that path.
- `list` prints one line per linked worktree under its repo, with branch and
  state, and prints nothing with exit 0 for a repo with no worktrees.
- `list <repo-path>` scopes to that repo only.
- For each of `clean`, `dirty`, `unpushed`, run `cleanup` on the same worktree
  and assert the outcomes agree: `clean` succeeds, the other two are refused.
- `list` reports `prunable` after the worktree directory is removed with
  `rm -rf`, and the repo's worktree count is unchanged by running `list`.

Negative paths, all required:

- **The in-place flow stays frictionless.** `begin` with no `-d` on an
  unprotected repo exits 0 and prints the repo path (`:254-256`, kept, renamed),
  and additionally: the repo still has exactly one worktree record afterwards,
  no `agent/*` branch was created, and `$HOME/changes` does not exist. Then
  edit, `record -f`, and assert the commit and push succeed as they do at
  `:301-312`.
- The outside-`$HOME` no-remote repo with no `-d` still passes through
  (`:258-262`, kept, renamed).
- `begin` with no `-d` on a protected repo still exits 1 and the message still
  contains `-d` (`:248-250`, kept; assertion text unchanged so the reworded
  message is constrained to keep the substring).
- **Duplicate description fails loud on an unprotected repo**, both ways: exit 5
  and "already exists locally" after a first `begin -d`, and exit 5 and "already
  exists on origin" for a branch pushed to the remote beforehand. These mirror
  `:264-279` and prove the existing rails now cover the flipped path.
- `begin -d` on a repo with no `origin` remote exits 1 with a message naming the
  missing remote, not the `ls-remote` failure.
- `begin -d` on a repo that has `origin` but no `refs/remotes/origin/HEAD` and
  no `protected-branches` entry exits 1 and names `remote set-head`. This is the
  `workspace_repo_default_branch` landmine; without this test the empty-base
  path is unexercised.
- Invalid descriptions still exit 1 on an unprotected repo, not only a protected
  one (`:291-297`).
- `begin -d` does not move the shared checkout: after a successful call, assert
  the main checkout is still on its default branch with a clean status.

Harness changes: drop the `/tmp` sweep at `tests/allod-change.sh:20`, which is
dead once worktrees land under the temporary `HOME` the harness already removes.

## Rollback Plan

Revert the single `allod/tools` commit. Before the human rebuilds there is
nothing deployed to undo; after it, the revert needs a second rebuild to take
effect, and until then `begin -d` on an unprotected repo keeps creating
worktrees, which is inconvenient rather than harmful.

Worktrees already created under `~/changes` survive the revert as valid linked
worktrees. They stay reachable through `git worktree list`, `record` and
`submit` operate on `$PWD` and work inside them unchanged, and `cleanup`
reclaims them. The C3 handoff file becomes inert, and it needs no cleanup path
because `git worktree remove` deletes the private git dir that holds it.
Recover any uncommitted work from a worktree before pruning it.

Partial states:

- If `begin` fails between `worktree add` and the handoff write, its own
  rollback has already removed the directory and pruned the admin entry. A
  leftover `agent/<description>` branch is possible and is reported rather than
  deleted; clear it with `git -C <repo> branch -D agent/<description>` after
  confirming no other agent owns it.
- If the `allod/memory` commit landed and the tools change is reverted, revert
  the memory commit too. It is a separate commit in a separate repo precisely so
  the two can move independently.
- `~/changes` itself is left in place on revert. It is an empty directory at
  worst and holds recoverable work at best; removing it is never part of a
  rollback.
