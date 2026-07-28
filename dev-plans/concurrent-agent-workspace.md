# Concurrent Agent Workspace Safety

## Tracking Issue

https://forge.anarch.diy/allod/tools/issues/115

Five implementation issues, each its own PR in `allod/tools`, plus a possible
`allod/archetypes` PR for hook deployment under allod/tools#112. Each PR uses
`Closes allod/tools#<child>` for its own issue and `Refs allod/tools#115`; the
last PR to land carries `Closes allod/tools#115`.

This document is the arc's shared home. It owns the contracts every child must
agree on, the sequencing between them, and the review depth each one gets. It
does not own implementation detail: a child carrying enough of it gets its own
dev plan, as allod/tools#116 did in
`archive/dev-plans/allod-change-always-isolate.md`.

## Goal

Make several coding agents sharing one dev VM a safe configuration, so a second
agent cannot move a first agent's checkout or sweep its edits without a loud
refusal.

## Scope

In scope:

- The four cross-issue contracts in Interface Contracts, which allod/tools#112,
  allod/tools#116, allod/tools#117, allod/tools#118, and allod/tools#119 must
  all implement identically.
- Sequencing and dependency order across those five issues.
- Per-issue residual risk and review depth.

Out of scope:

- Implementation detail for any single child; each child's issue body, and for
  allod/tools#116 a child dev plan, owns that.
- Agent identity as a general feature, and per-VM agent topology — both already
  excluded by allod/tools#115's own scope.
- Changing forge-side branch protection rules, which is admin-scoped and human
  only (see Agent Gates).

Known gap, owned by no child: nothing verifies that an *unprotected* shared
checkout is actually on its default branch before an in-place commit.
`refuse_protected_branch_record` (`allod:230`) only fires when HEAD equals a
*protected* branch, allod/tools#118's moved-checkout guard is deliberately a
no-op when `begin` was never called, and allod/tools#119's hook rule covers only
the `agent/*` prefix. A shared checkout parked on some other branch therefore
commits there silently. With allod/tools#116 landed, branch work no longer lives
in shared checkouts, and allod/tools#119 narrows the residue further, but
nothing in this arc closes it. Raise it as its own issue if a checkout is ever
found parked on a stray branch.

## Risk Assessment

Residual risk: R3 High for the arc, carried almost entirely by the two issues
that change `git-hooks/protected-refs-policy`. Every commit and push in every
repo on the VM passes through that hook, so a defect there either blocks all
work or silently reopens the hole it was meant to close. The tool-side issues
are individually R1–R2.

Per-issue triage. Review depth is set by the blast radius of the change itself,
not by the severity of the bug it fixes:

| Issue | Risk | Review depth | Reason |
|---|---|---|---|
| allod/tools#118 record refusals | R1 | One review pass, different model | Two pure refusals that preflight and touch nothing on failure |
| allod/tools#124 concurrent push | R2 | Deferred behind a decision criterion | Rebase state machine; safe but not shown to be worth building |
| allod/tools#119 hook refusal | R3 | Dev plan + one review pass, different model | Deepest enforcement layer, but a small rule with a 200-line test harness already in place |
| allod/tools#112 remote-identity rails | R3 | Dev plan + review cycle to convergence | Security boundary, cross-repo packaging decision, human-gated validation |

allod/tools#118 originally carried three mechanisms and was triaged at R2–R3
with a convergence cycle. It was split two ways, and both halves got cheaper.

The auto-rebase moved to allod/tools#124, deferred behind a decision criterion
rather than dropped. An earlier revision of this document claimed it could
reintroduce the arc's own failure class by rewriting history in a shared
checkout under a concurrent writer. Fixture testing refuted that: when another
agent has uncommitted work, `git pull --rebase` refuses outright and touches
nothing, and the one bad state — conflict, rebase-in-progress, detached HEAD,
markers in the shared tree — is fully recoverable by `git rebase --abort`. It
is safe to build. What it is not is demonstrably worth building, since the
rejected push already fails loud under `set -euo pipefail` and recovery is one
documented command. Note the fail-closed property holds only while
`rebase.autoStash` is false, so an implementation must pass
`-c rebase.autoStash=false` rather than inherit config.

What remains in allod/tools#118 is two pure refusals, and the friction problem
that drove the convergence-cycle rating dissolved rather than being accepted. No
mechanism can attribute a tracked-file edit to an agent, so the sweep guard can
only be positional — named versus unnamed. But the hazard is not uniform: it
exists only in a shared checkout, because a linked worktree belongs to one agent
by construction. Scoping the refusal to a main checkout keeps `git add -u` and
zero friction everywhere allod/tools#116 sends branch work, and costs the
in-place flow one flag per file, which `allod/memory` `git-workflow.md` already
instructs agents to pass. That is principle 14 promoting a prose rule into the
tool, with the override staying standard git per principle 4.

Why allod/tools#112 needs one despite a smaller live bug than its body claims:
the change touches the rail every commit passes through, must decide where a
helper shared between a CLI and a standalone hook file lives (see Agent Gates),
and cannot be fully validated by the agent. Its severity, separately, is lower
than the issue body states — see Forge-Side Protection below.

Human scrutiny:

- The hook diffs, first and hardest: confirm the near-miss and worktree rules
  fail closed and do not over-fire on genuinely unlisted repos.
- allod/tools#118's staging refusal: check the failure message names the
  offending paths and that the in-place flow is genuinely unchanged.

### Forge-Side Protection

allod/tools#112's body says a path-mismatched checkout lets an agent "commit and
push straight to `master` ... with no refusal and no hook block". The commit half
is verified. The push half is not: the forge protects the default branch
server-side for the framework repos, so the push is rejected regardless of what
the local rails do. The fail-open is a local-state hazard and a defence-in-depth
gap, not a path to unauthorised publication. This lowers allod/tools#112's
urgency; it does not lower its review depth, which comes from the blast radius
of the change itself.

The same finding removes the arc-level ordering constraint. allod/tools#112 was
argued as a prerequisite for the whole arc because unconditional isolation would
move every agent change into an unguarded zone. Inside a linked worktree on an
`agent/*` branch, that zone is empty of live protection today:
`allowed-external-remotes` and `active-pr-branches` are keyed off the remote URL
and still fire, the `agent/*` force-push block is keyed off the branch name and
still fires, `signing-required-branches` is currently an empty file, and the
`protected-branches` check is unreachable because git refuses to check out the
protected branch in a second worktree while the main checkout holds it. The gap
is latent, activating the day a branch is added to `signing-required-branches`.
One live consequence is worth recording: `run_repo_hook`
(`git-hooks/protected-refs-policy:98`) looks for `$repo/.git/hooks/$hook`, and a
worktree's `.git` is a file, so repo-local hooks do not run in a worktree.
`run_tracked_hook` reads tracked paths and is unaffected.

## Interface Contracts

Five facts are shared across the children. They are fixed here so the children
cannot drift; a child that needs to change one changes it here first.

**C1 — Worktree siting.** `allod change begin -d` creates worktrees at
`~/changes/<slug>-<description>-XXXXXX`, implemented in allod/tools#116. Never
under `$WORK_DIR`: a worktree there is collected as a repo by
`workspace_collect_repos`, and its `$HOME`-relative path is a lookup key
matching nothing. No consumer may depend on this path — enumerate via C2
instead. Anything still under the old `/tmp` siting is on a hidden deadline:
`/etc/tmpfiles.d/tmp.conf` carries `q /tmp 1777 root root 10d` and
`systemd-tmpfiles-clean.timer` is active, so a stalled agent's uncommitted work
there is destroyed at ten days rather than merely left lying around.

**C2 — Worktree detection and repo identity.** One mechanism everywhere, all
verified:

- A directory is a linked worktree when `git rev-parse --path-format=absolute
  --git-dir` differs from `--git-common-dir`.
- The owning repo is `dirname` of the common dir. This is already
  `main_repo_dir_for_dir` (`allod:111`); the hook must adopt the same rule under
  allod/tools#112.
- Enumeration is `git -C <repo> worktree list --porcelain`, never a filesystem
  glob. Its first record is the main checkout itself, which consumers skip.

Owners: allod/tools#112 (hook), allod/tools#116, allod/tools#117,
allod/tools#119.

**C3 — Branch handoff.** `begin` writes the branch it created to
`$(git rev-parse --path-format=absolute --git-dir)/allod-change-branch` inside
the new worktree; `record` reads the same path and refuses when it disagrees
with current HEAD. Verified properties: in a linked worktree `--git-dir` is the
private `<main>/.git/worktrees/<name>`, so the file is per-worktree rather than
per-repo, and `git worktree remove` deletes it, so it needs no cleanup path of
its own. A missing file means `begin` was never called and `record` proceeds
unchanged, which is what keeps the in-place default-branch flow unaffected.
`begin` writes it as of allod/tools#116; allod/tools#118 owns the reader.

**C4 — What `-d` means.** After allod/tools#116, `-d <description>` is the
isolation switch, not a protected-repo formality:

- `-d` present: create a worktree and `agent/<description>` for every repo,
  protected or not.
- `-d` absent: print the shared checkout path and create nothing — the in-place
  default-branch flow for memory and planning docs.
- `protected-branches` goes back to governing only which branch is protected.

Omitting `-d` on a protected repo fails loud, keyed on the repo rather than on
HEAD, because committing in place to a protected branch is refused later anyway
(`refuse_protected_branch_record`) and failing at `begin` is louder and earlier.
The distinction that resolves it: C4 removes protection as the *isolation*
switch, not as a *validation* input. The full argument is in
`archive/dev-plans/allod-change-always-isolate.md`.

**C5 — The collector's output set is frozen.** `workspace_collect_repos` keeps
returning exactly the registry checkouts. allod/tools#117 added
`workspace_collect_worktrees` as a second entry point rather than changing it,
because `pull-all`, `flake-status`, and `flake-update-cascade` all mutate based
on its output. That constraint binds every later consumer too.

Do not unify worktree enumeration across `work-diff` and `allod change list`.
They have opposite requirements: `work-diff` wants worktrees with content to
show, so allod/tools#117's `workspace_collect_worktrees` deliberately skips
prunable entries and does not parse lock state, while `list` wants everything
still registered plus the reason it is not actionable. Recovering `prunable` and
`locked` needs a second porcelain parse regardless, so sharing buys nothing.
`list` inlines its own enumeration; this is settled, not a sequencing accident.

Coordination hazard in the same file: `workspace_repo_default_branch` has a
latent bug of the kind `allod/memory` `shell.md` warns about, and an earlier
revision of this contract described it backwards. Its `|| echo "master"` binds
to the pipeline, so the behaviour depends on the caller's shell options: without
`pipefail` the status is `sed`'s and the fallback is unreachable, returning
empty; with it the status is git's and the fallback fires, silently guessing
`master`. Every consumer sets `set -euo pipefail`, so the live behaviour is the
guess, not the empty return. Verified.

allod/tools#116 and allod/tools#117 both landed without touching the helper.
allod/tools#116 defends itself instead, asserting `symbolic-ref
refs/remotes/origin/HEAD` resolves before consuming it, so `begin -d` cannot
branch a repo off a guessed base. The helper itself is issue allod/tools#126;
its remaining consumers still take the guess.

## Sequencing

allod/tools#117 and allod/tools#116 have merged, so C1, C2, C3, and C4 are all
implemented and nothing downstream is blocked on them.

```
allod/tools#118  (unblocked — C2 and the C3 handoff both exist)
allod/tools#112  (independent)
   └── allod/tools#119  (needs the hook to resolve identity from a worktree)

allod/tools#124  (independent — deferred behind its decision criterion)
```

allod/tools#118's two refusals were split by dependency while C3 was pending:
the sweep guard needs only C2 detection (`main_repo_dir_for_dir`), and the
moved-checkout guard reads the C3 handoff. Both inputs exist now, so the issue
can be taken whole.

Only allod/tools#119 has a hard dependency, and only on allod/tools#112. The
arc-wide "112 must land first" constraint does not survive the Forge-Side
Protection finding above.

Order: allod/tools#118 next. allod/tools#112 and allod/tools#119 form a second
wave that can run in parallel with it, since they share no files.
allod/tools#124 sits outside the order entirely until its criterion is met.

## Agent Gates

- **Hook deployment.** `git-hooks/protected-refs-policy` reaches a VM through
  `allod/archetypes/modules/agent-hooks.nix:9-11`, which copies it to
  `~/.config/git/protected-refs-policy`. Installing a changed hook needs a
  home-manager rebuild, which is host-side. The agent validates against
  `tests/git-hooks/protected-refs-policy.sh` with a fixture `core.hooksPath`,
  never against the live installed hook, and the human confirms post-rebuild
  behavior. Blocks: final verification of allod/tools#112 and allod/tools#119.
- **Shared-helper packaging.** The hook is deployed as a single standalone file
  with no sibling `lib/`, so allod/tools#112's "one shared helper" goal needs a
  decision: a second `home.file` entry in `allod/archetypes`, an absolute store
  path reference, or accepted duplication with tests on both copies. The first
  two make it a cross-repo change requiring a second PR and a human merge.
- **Forge-side branch protection.** Reading the rules needs repo admin; the
  agent token gets 403 on `/api/v1/repos/{owner}/{repo}/branch_protections`.
  Agents can read effective status per branch but cannot enumerate or change the
  rules. Any change to forge-side protection is a human act at the forge.

## Acceptance Tests

Per child, all runnable by the agent:

```sh
# allod/tools#118 — CLI contract and the record guards; also the regression
# suite for the C1-C4 behaviour already landed
bash tests/allod-change.sh

# allod/tools#112, allod/tools#119 — near-miss fixture, worktree fixture,
# refusal, default-branch pass-through, and the human override
bash tests/git-hooks/protected-refs-policy.sh
```

Each child adds negative cases, not only happy paths: a genuinely unlisted repo
must stay unprotected under allod/tools#112, an in-place default-branch commit
must stay frictionless under allod/tools#118, and a human `--no-verify` must
still pass under allod/tools#119.

## Rollback Plan

Every child is a straight revert of one `allod/tools` commit; none writes
persistent state outside a git worktree, and C3's handoff file dies with the
worktree that holds it.

Two qualifications:

- Hook changes reach a VM only through a rebuild, so reverting allod/tools#112
  or allod/tools#119 in the repo is not enough — the human rebuilds to restore
  the previous hook. Until then, `git commit --no-verify` is the standard
  override, per architecture principle 4.
- allod/tools#116 has shipped, so reverting it now leaves any worktree already
  under `~/changes` orphaned rather than broken. They stay valid git worktrees
  reachable through `git worktree list`, and `allod change list` names each one
  with what blocks its removal; recover any uncommitted work before pruning.
