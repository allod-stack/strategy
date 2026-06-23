# User Stories: Agentic Development Flow

Requirements analysis for notes #36.

## Context

Long coding sessions cause agents to forget git workflow rules by the time they reach commit/push/PR steps. Issue #36 proposes a dedicated subagent. Before designing the solution, this document maps the full agentic development lifecycle from the user's perspective to nail down what the tool actually needs to handle.

## The User as Orchestrator

The user manages multiple agent sessions and their outputs. Their job is: assign work, review plans, give feedback on PRs, merge, and handle agent gates. They are not executing git commands — agents do that. The user's tools for situational awareness:

- `work-diff` — shows dirty state across all repos
- `pull-all` — syncs all repos at session start
- `forge pr list` / `forge issue list` — shows open work on Forgejo
- Direct conversation with agent — real-time steering
- PR comments — async feedback

### User-Initiated Actions

1. Check workspace state (`work-diff`, `pull-all`)
2. Launch agent session and point it at work (issue number, description)
3. Review dev plan output and give feedback
4. Approve plan (agent proceeds to implementation)
5. Review PR (run Validation commands, read diff)
6. Give feedback on PR (direct or via forge comment)
7. Merge PR
8. Handle agent gates (deploy, re-encrypt, provision — host-only actions)
9. Open follow-up issues when post-merge problems surface

### Feedback Mechanisms

1. **Plan review loop** — Review agent iterates on plan file. User monitors, can inject focus areas or override decisions. Converges to 0 BLOCKERs/GAPs/QUESTIONs across multiple passes (real example: token rotation took 8 passes, SSH key rotation 2+ passes ongoing).
2. **PR review comments** — Read-only review subagent comments findings. Implementation agent fixes in followup commit.
3. **User PR feedback** — User runs Validation commands from the PR description, then tells agent what's wrong. Agent records the feedback as a PR comment before implementing the fix.
4. **Direct conversation** — User steers the agent mid-session.
5. **Follow-up issues** — Post-merge discoveries feed back as new issues.

## Development State Machine

```
IDLE
  |  user assigns issue
  v
EXPLORING -----> PLANNING
                    |  plan draft complete
                    v
               REVIEWING -.
                 |  ^     | review pass finds issues
                 |  '-----' agent commits fixes, rotates reviewer
                 |  0 BLOCKERs/GAPs/QUESTIONs
                 v
            IMPLEMENTING
                 |  code complete, tests pass
                 v                                       user
            PR_OPEN  <---.  followup     MERGED -------> gives
                 |       |  commit          ^            feedback
                 v       |                  |               |
            ITERATING ---'              user merges     FOLLOWUP
              (review feedback            PR              |
               or user feedback)                     new issue opened
                                                         |
                                                    back to EXPLORING
```

### Git Ceremony Touchpoints

These are the moments where an agent interacts with git and forge. Each is a potential failure point if the agent has lost track of the workflow rules.

- **C1: Plan commits** — During REVIEWING. Agent commits plan fixes to notes (non-protected, direct to master). Low risk, but still needs: stage, commit, push, no force-push.

- **C2: First implementation commit** — Transition from IMPLEMENTING to first commit. Highest risk. Must check protected-branches, create branch if needed, stage correct files, commit, push. For protected repos, also needs worktree isolation.

- **C3: PR creation** — Often same session as C2. Must assemble title, body with Validation section, base branch. Uses `forge pr create`. Easiest to forget Validation section here.

- **C4: Followup commit after review** — Agent already on branch. Must add commit (never amend), push (never force). Risk: agent amends because "it's just a small fix."

- **C5: Followup commit after user feedback** — Same as C4, plus: must comment on PR recording the feedback before implementing.

- **C6: Multi-repo: subsequent PRs** — Agent moves to next repo in dependency order. New branch, new PR, with `Depends on:` note. Risk: forgetting dependency ordering, using wrong base branch.

## Scenario A: Single-Repo Feature

**Setup:** User wants `forge issue close` subcommand added to allod/tools.

### Phase: Planning (brief — small feature, no formal review)

```
User: "implement forge issue close"
Agent: reads forge script, understands pattern
Agent: plan is simple enough to skip formal review
```

### Phase: Implementation

```
Agent: checks ~/.config/git/protected-branches
       → work/allod/tools is listed → protected
Agent: needs isolation (worktree or equivalent)
Agent: creates agent/forge-issue-close branch from master
Agent: codes the feature, writes test
Agent: runs tests locally
```

**[C2+C3] First ceremony — branch + commit + PR:**
```
- Stage: forge (modified), tests/forge/issue-close.sh (new)
- Commit: "add forge issue close subcommand"
- Push: agent/forge-issue-close → origin
- PR: title="add forge issue close"
       body includes ## Validation with: forge issue close <test-issue>
       base=master
```

### Phase: Review feedback

```
Review subagent: "close should accept issue URLs, not just numbers —
                  forge already parses URLs in pr view"
```

**[C4] Second ceremony — followup commit:**
```
- Stage: forge (modified), tests updated
- Commit: "accept issue URLs in forge issue close"
- Push: agent/forge-issue-close → origin
- No PR creation (PR already exists)
```

### Phase: User feedback

```
User runs validation: forge issue close 99
User: "works, but it should print confirmation, not silence"
```

**[C5] Third ceremony — followup after user feedback:**
```
- Agent comments on PR: "User requests confirmation output on close"
- Stage: forge (modified)
- Commit: "print confirmation after issue close"
- Push: agent/forge-issue-close → origin
```

### Phase: Merge

User approves. Merges via Forgejo UI. Runs `pull-all`.

**Ceremony count:** 3 touchpoints. Branch creation only at first.

---

## Scenario B: Multi-Repo Feature with Plan Review

**Setup:** User wants credential rotation tooling. Touches secrets (data format), nexus (rotation script), and profiles (validation + flake lock). All three repos are protected.

### Phase: Planning

```
User: "implement credential rotation from notes plan"
Agent: reads plan, begins formal review cycle
```

**[C1] Plan review commits (notes, non-protected):**

Review passes commit directly to notes master. Each pass:

```
Pass 1: 3 GAPs, 1 SIMPLIFY → agent commits 3 fixes to plan file
Pass 2: 1 BLOCKER (credential-store format would corrupt .git-credentials),
        2 GAPs → agent commits fixes, blocker requires interface redesign
Pass 3: 2 BLOCKERs surface (locked flake deployment order; missing Forgejo
        SSH keys) → fixes committed
        ...pattern continues. new blockers surface when fixing gaps.
Pass N: 0 BLOCKERs, 0 GAPs, 0 QUESTIONs → plan approved
```

Each review pass: stage plan file changes → commit → push to master. Simple ceremony, but must happen reliably 20-30 times across the full review cycle.

**Agent rotation between passes** means different agent sessions handle different passes. Each session's first action is reading the plan and the review prompt. The ceremony per commit is identical.

### Phase: Implementation (PR 1 of 3 — secrets)

The plan specifies ordering: secrets first, then nexus, then profiles.

```
Agent: checks protected-branches → secrets is protected
Agent: isolates (worktree), creates agent/credential-rotation branch
Agent: implements machine-host-keys.json, credentials.nix changes, validation
Agent: runs nix flake check in secrets
```

**[C2+C3] Ceremony — secrets PR:**
```
- Stage: machine-host-keys.json, credentials.nix, secrets.nix, flake.nix
- Commit: "add credential inventory and rotatable key data format"
- Push: agent/credential-rotation → origin
- PR: title="credential inventory and key data format"
       body includes ## Validation with: cd secrets && nix flake check
       base=master
```

Review subagent finds issues.

**[C4] Followup ceremony:**
```
- Commit: "fix credential-store payload format"
- Push (no force)
```

**AGENT GATE: Human merges secrets PR and confirms nix flake check passes on merged master.**

### Phase: Implementation (PR 2 of 3 — nexus)

Agent gate cleared. Agent starts on nexus.

```
Agent: isolates, creates agent/rotate-token branch in nexus
Agent: implements scripts/rotate-token, adds to flake.nix
Agent: runs nix flake check in nexus
```

**[C2+C3+C6] Ceremony — nexus PR with dependency:**
```
- Stage: scripts/rotate-token, flake.nix
- Commit: "add rotate-token script"
- Push: agent/rotate-token → origin
- PR: title="add rotate-token script"
       body includes ## Validation AND "Depends on: secrets#N"
       base=master
```

**AGENT GATE: Human merges nexus PR, rebuilds nexus so rotate-token is on PATH.**

### Phase: Implementation (PR 3 of 3 — profiles)

```
Agent: isolates, creates agent/credential-validation branch in profiles
Agent: updates profiles to consume merged secrets + nexus
       nix flake update secrets nexus
Agent: updates validation code, runs nix flake check
```

**[C2+C3+C6] Ceremony — profiles PR with dependencies:**
```
- Stage: validation changes, flake.lock
- Commit: "update credential validation for rotation support"
- Push: agent/credential-validation → origin
- PR: title="credential validation for rotation"
       body includes ## Validation AND "Depends on: secrets#N, nexus#M"
       base=master
```

**AGENT GATE: Human merges profiles PR and deploys.**

### Phase: Post-"done" feedback

User deploys and tests rotate-token manually on host.

```
User: "rotate-token stage works but the verification output is wrong —
       it prints the old key fingerprint, not the new one"
```

This requires a NEW PR to nexus (the "finished" feature needs fixing):

```
Agent: isolates nexus, creates agent/fix-rotate-token-verify branch
Agent: fixes verification output
```

**[C2+C3] Ceremony — follow-up fix PR:**
```
- Stage: scripts/rotate-token
- Commit: "fix verification output to show new key fingerprint"
- Push
- PR: title="fix rotate-token verification output"
       body includes ## Validation with verification commands
       base=master
```

Then profiles needs another flake lock update PR to pull the fixed nexus.

**Ceremony count:** ~8 git ceremony touchpoints across 4 PRs + ~25 plan review commits. Dependency tracking (`--depends-on`) is real and needed.

---

## Scenario C: Concurrent Sessions

**Setup:** User has two agents working on different features in allod/tools simultaneously.

```
Terminal 1: Agent A → "add git-ceremony script to tools"
Terminal 2: Agent B → "fix forge pr create base branch default"
```

### Without isolation (the failure case)

Both agents are in `/home/vnprc/work/allod/tools`. Agent A creates `agent/git-ceremony`, starts editing. Agent B tries to create `agent/forge-base-default` but the working tree has Agent A's uncommitted changes. `git checkout -b` refuses or clobbers files. Chaos.

### With worktree isolation

```
Agent A: creates worktree → .claude/worktrees/<uuid-a>/
         creates agent/git-ceremony branch
         works in isolated directory

Agent B: creates worktree → .claude/worktrees/<uuid-b>/
         creates agent/forge-base-default branch
         works in isolated directory
```

Both finish independently. Both run git-ceremony. Both create separate PRs. No collision.

**Key insight for git-ceremony scope:** The tool can either:
(a) Manage worktree lifecycle itself (`git worktree add/remove`), or
(b) Assume the caller already isolated (Claude Code's EnterWorktree, or a custom harness)

Option (a) makes the tool more self-contained. Option (b) keeps it simpler and avoids duplicating harness logic. Both are valid — depends on how much the tool should own.

### Branch name collision risk

If two agents both use `-d "fix-bug"`, they'd both try to create `agent/fix-bug`. Mitigation options:
- Script appends a short random suffix: `agent/fix-bug-a3f2`
- Script detects existing remote branch and fails with an error
- Caller is responsible for unique descriptions

---

## Scenario D: Post-Merge Discovery

**Setup:** git-ceremony has been merged to allod/tools. User uses it in practice.

```
User: agent uses git-ceremony in nexus repo
      Agent is on a detached HEAD (mid-way through nix flake update)
      git-ceremony fails: "not a branch" error with no useful message
User: opens notes issue #42
```

New agent session:

```
Agent: reads issue #42, explores git-ceremony source
Agent: small fix, no formal plan review needed
Agent: isolates tools repo, creates agent/fix-detached-head branch
Agent: adds detached HEAD detection + useful error message + test
```

**[C2+C3] Ceremony:**
```
- Stage: git-ceremony (modified), tests/git-ceremony.sh (modified)
- Commit: "detect detached HEAD and give actionable error"
- Push
- PR with Validation section
```

User reviews, merges. The tool that failed is fixed by the same workflow it supports.

**Insight:** Git-ceremony's own error messages matter. When it refuses an operation, the message should tell the agent (or human) exactly what to do instead.

---

## Derived Requirements

### What git-ceremony must handle

From the scenarios above, every touchpoint needs some subset of:

| Capability | C1 (plan) | C2 (first commit) | C3 (PR) | C4 (followup) | C5 (user fix) | C6 (multi-repo) |
|---|---|---|---|---|---|---|
| Check protected-branches | - | yes | - | - | - | yes |
| Create branch | - | if protected | - | - | - | if protected |
| Stage files | yes | yes | - | yes | yes | yes |
| Commit (never amend) | yes | yes | - | yes | yes | yes |
| Push (never force) | yes | yes | - | yes | yes | yes |
| Create PR via forge | - | - | yes | - | - | yes |
| Validate ## Validation | - | - | yes | - | - | yes |
| Add depends-on note | - | - | - | - | - | yes |
| Comment on PR | - | - | - | - | yes | - |

The first five rows (check, branch, stage, commit, push) fire at almost every touchpoint. The PR rows fire less often.

### What git-ceremony must refuse

1. Direct commit to a protected branch (force branch creation)
2. `--amend` on any commit (not even exposed as a flag)
3. `--force` or `--force-with-lease` on any push (not even exposed)
4. PR creation without `## Validation` in the body
5. Staging when nothing changed (exit with clear message)

### What git-ceremony must not do

1. Worktree management (belongs to the agent harness — Claude Code, custom FOSS harness, etc.)
2. Plan review orchestration (that's the agent's job)
3. Multi-repo sequencing in a single invocation (caller handles ordering)
4. Merge PRs (user action)
5. Guess — if the state is ambiguous (wrong branch, detached HEAD, dirty unrelated changes), fail with a clear message

### Must-have error messages

From Scenario D: when git-ceremony refuses an operation, the error message should say what the problem is AND what to do about it. Examples:

```
git-ceremony: allod/tools master is protected; use -d <description> to create an agent branch
git-ceremony: already on agent/foo — adding commit (use a new -d to start a fresh branch)
git-ceremony: PR body is missing a ## Validation section — add one with concrete test commands
git-ceremony: nothing to commit (working tree clean)
git-ceremony: detached HEAD — checkout a branch first or use -d to create one
git-ceremony: refusing push — git-ceremony never force-pushes
```

## Design Decisions (resolved)

1. **Worktree scope.** git-ceremony owns the worktree lifecycle. This makes the tool self-contained — a FOSS harness on pi+openrouter calls `git-ceremony begin` and gets isolation for free, no need to reimplement EnterWorktree.

2. **PR commenting.** Separate from git-ceremony. The agent calls `forge pr comment` directly. git-ceremony owns the mechanical git+forge-PR pipeline, not the conversational parts. Keeps the tool focused.

3. **Commit-only vs PR mode.** Single `ship` subcommand. Body-presence triggers PR creation: pass `-b`/`-B` to create a PR, omit them for commit+push only. `--commit-only` is a safety override that skips PR even if a body is present.

4. **Branch reuse.** When already on `agent/*`, skip branch creation. Don't query forge for open PRs — that adds latency and complexity. The agent is responsible for knowing whether it wants a new branch or to continue on the current one.

