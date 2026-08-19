# Bash-to-Go Migration for Program-Shaped Tools

## Tracking Issue

https://forge.anarch.diy/allod/tools/issues/98

Multi-phase, multi-repo work. All implementation PRs use `Refs allod/tools#98`.
The final Phase 4 cleanup PR in `allod/tools` carries `Closes allod/tools#98`.

## Goal

Replace the four program-shaped bash tools in `allod/tools` (`forge`, `allod`,
`flake-update-cascade`, `flake-status`) with stdlib-only Go binaries that keep
the same CLI contracts, while everything pipeline-shaped stays in bash.

## Language Decision

Go, standard library only. `vendorHash = null` is enforced in Nix packaging so
a third-party Go dependency fails the build; adding one requires a plan
amendment.

Why Go over the alternatives, against the Allod priorities:

- **Minimalism** - one static binary per tool, no runtime interpreter, no
  module supply chain. `net/http` + `encoding/json` replace the runtime
  `curl` + `jq` dependency for `forge`. Rust CLIs realistically pull
  clap/serde/reqwest (hundreds of transitive crates); Python adds an
  interpreter runtime that is already policy-banned on privacy VMs.
- **Security** - real data structures and typed errors remove the injection,
  quoting, and word-splitting bug class that dominates the fix history
  ("grep injection", "glob safety", "Quote patch HEAD revspecs"). API tokens
  stay in-process: no argv, no child processes, strictly better than the
  `curl --config` file-descriptor workaround.
- **Usability** - identical CLI surfaces; single binaries; error messages
  built from typed failures instead of whichever pipeline stage broke.
- **Maintainability** - `go test` with `httptest` replaces ~4,700 lines of
  hand-rolled bash mocks (including a fake Forgejo API inside a mock `curl`);
  `gofmt` normalizes diffs; agents author and review Go reliably.
- **Simplicity** - Go is boring and explicit. The verbosity of `if err != nil`
  is the honest version of what `set -e` plus manual `set +e` toggling hides
  today.

Decision rule for the split: migrate when a tool owns data structures
(JSON, manifests, registries), implements a protocol or API client, or runs a
multi-mode state machine with rollback. Stay in bash when a tool is thin
fork/exec orchestration of git/ssh/nix/virsh, a git hook, or trivial glue.

Applied to the inventory:

| Tool | Lines | Decision | Reason |
|---|---|---|---|
| `forge` | 2505 | Go (Phase 1) | REST client: JSON, pagination, auth |
| `allod` | 1235 | Go (Phase 2) | patch transfer protocol, manifest integrity, change workflow exit-code contracts |
| `flake-update-cascade` | 486 | Go (Phase 3) | three-mode state machine, rollback paths, duplicated mode blocks |
| `flake-status` | 490 | Go (Phase 3) | parallel aggregation via tempfile IPC → goroutines and structs |
| `pull-all`, `work-diff` | 200 | bash | pipeline-shaped; active redesign in allod/tools#87 stays bash |
| `setup-tracked-hooks`, `lib/workspace.sh` | 91 | bash | thin glue; lib still needed by staying tools |
| `git-hooks/protected-refs-policy` | 200 | bash | enforcement boundary; ubiquity and zero build step outweigh rewrite gains |
| `nexus` provisioning scripts | ~620 | bash | virsh/nixos-anywhere/ssh orchestration; R4 first-boot paths, human-only validation |
| `nexus` rotation suite (`rotate-token` et al.) | ~3175 | deferred | program-shaped but host-only, secrets-touching, human-only validation; revisit after Phase 3 via follow-up issue |
| `vm`/`profiles` hooks and glue | ~60 | bash | trivial |

## Scope

In scope:

- `allod/tools`: Go module (`go.mod`, `cmd/<tool>/`, `internal/`), Go
  implementations of the four tools, a dev-only `flake.nix` for local builds
  and checks, parameterizing existing bash test suites to accept a
  binary-under-test path, and eventual deletion of migrated bash sources.
- `allod/archetypes`: `modules/dev-home-shared.nix` packaging switches
  (writeShellApplication → buildGoModule per tool per phase). The dev-VM
  wrapper moved here in the archetypes/profiles split; no Go toolchain is
  added to VM packages (see the 2026-08-19 amendment under Sequencing).
- `allod/nexus`: `nix/home.nix` same packaging switches for the host.
- `allod/memory`: `vm-tooling.md` policy line for the Go toolchain;
  `allod.md` tool notes if command behavior notes change.

Out of scope:

- Any behavior change to migrated tools beyond enumerated known-bug fixes
  (see Interface Contracts). Feature work on a tool freezes during its phase;
  open feature issues (allod/tools#89, #90, #91) land before or after, not
  during.
- `pull-all` and `work-diff` (owned by allod/tools#87), all git hooks, all
  `nexus` provisioning and rotation scripts, `vm`/`profiles` glue.
- Privacy VMs: no toolchain, no tool changes, nothing lands there.
- Forgejo server, secrets content, inventory data.

## Risk Assessment

Residual risk: R3 High overall, because cutovers swap generated VM behavior
for the tools agents use daily (`forge`, `allod change`), and validation of
the deployed wrappers depends on human-gated rebuilds.

Why:

- The bash originals stay in-tree and the profiles/nexus wrappers are a
  one-line revert per tool, so rollback is a pin revert plus rebuild.
- Exit codes and stdout contracts are load-bearing (agents script against
  `allod change begin` output and `die` codes; `flake-update-cascade` and
  `allod change submit` shell out to `forge pr find-by-head`). A silent
  contract drift is the worst credible failure, and it would land on every
  dev VM at once after a rebuild.
- Existing `allod` and `flake-update-cascade` bash suites use exec-level
  mocking and real git fixtures, so they can run unchanged against the Go
  binaries; that is strong parity evidence the agent can produce locally.
- `forge` parity rests on ported scenario tests (the bash suite mocks `curl`,
  which the Go binary never execs), so Phase 1 carries the largest
  test-migration burden and gets a live read-only comparison gate.

Human scrutiny:

- Phase 1: token handling paths in Go `forge` (in-process only, no argv, no
  child processes, no logging of `Authorization`), and the httptest fixture
  fidelity against real Forgejo responses.
- Phase 2: `allod patch` remote protocol equivalence (SSH invocation shape,
  manifest schema, integrity failure exit codes 10-16).
- Cutover PRs: confirm wrapper diffs in archetypes/nexus touch only the
  intended tool and that the bash file is still shipped until Phase 4.
- Phase 4: deletion ordering (see Sequencing) so a profiles flake update can
  never reference a deleted bash path.

| PR or milestone | Risk | Reason | Human scrutiny |
|---|---|---|---|
| P0 scaffold (amended: no VM toolchain) | R1 | Inert repo scaffolding only; dev builds use the tools repo devShell | Policy line in memory |
| P0 test parameterization | R1 | Bash suites gain env-var binary override, default unchanged | Suites still green on bash |
| P1 forge in Go | R3 | Auth-bearing API client swap; all agent Forge ops | Token paths, scenario coverage vs docs/forge.md, live read-only diff |
| P2 allod in Go | R3 | Daily change workflow plus patch protocol | Exit codes, begin/record/submit output, patch integrity checks |
| P3 cascade in Go | R3 | Multi-repo mutation, force-with-lease push paths | Preflight/rollback parity in fixtures |
| P3 flake-status in Go | R2 | Read-only reporting | Output golden-diff vs bash |
| P4 cleanup | R1 | Deletes bash sources already off the deploy path | Ordering: consumers rebuilt before deletion merges |

## Interface Contracts

Command names, flags, env vars, exit codes, and machine-consumed stdout are
frozen at current behavior. That freeze is executable, not prose: a tool's
contract is what its existing suite asserts and its `--help` and `docs/`
document, and a port is done when those suites pass unchanged against the Go
binary — `ALLOD_UNDER_TEST`, `CASCADE_UNDER_TEST`, and for `forge` the ported
scenario tests, all listed under Acceptance Tests.

This section deliberately does not restate those surfaces. The tools stay under
active development throughout the migration: `allod change`'s contract already
moved under allod/tools#116, which added `~/changes` siting, a `change list`
subcommand, and a branch-handoff file, and issues allod/tools#53, #88, #89, #90,
and #100 each change `forge`'s. Any enumeration copied here is therefore stale
by the time its phase begins, and a stale copy is worse than a pointer because
it reads as authoritative — one source of truth per fact, `architecture.md`
principle 8.

What belongs here is what the suites cannot state:

- **Why these contracts are load-bearing.** Agents script against `allod change
  begin`'s stdout and against the `die` exit codes, and a cutover reaches every
  dev VM at once after a rebuild, so a silent drift is the worst credible
  failure of this migration rather than an inconvenience.
- **Cross-tool couplings no single suite covers.** `forge pr find-by-head`
  prints a bare PR number or nothing, and both `allod change submit` and
  `flake-update-cascade` parse that. `allod change submit` shells out to `forge`
  rather than embedding it, which is what keeps `allod` from ever touching a
  token. Breaking either one breaks a consumer whose tests do not run in the
  broken tool's suite.
- **A requirement on the new implementation, not a description of the old.** Go
  `forge` keeps API tokens in-process: never in child argv or env, never logged.
  It makes no child processes for API calls, which is what makes that
  enforceable statically rather than by inspection. `token verify` keeps reading
  candidate tokens from stdin only.
- Known-bug divergence rule: correctness bugs still open at port time are
  fixed in the Go version, covered by a new test, and enumerated in the PR
  body; everything else is bug-for-bug parity. allod/tools#83 (find-by-head
  head-filter) has since been fixed in bash, so the Go port inherits it via
  parity.
- Nix: profiles/nexus build migrated tools with
  `pkgs.buildGoModule { src = allod-tools; vendorHash = null; }` from the
  existing `flake = false` input, wrapping binaries so `git` (and for cascade
  `nix`, `forge`) are on PATH exactly as `runtimeInputs` pins them today.
  `go.mod` declares a Go version ≤ the pinned nixpkgs Go.
- The dev-only `flake.nix` in `allod/tools` must not change how consumers
  read the input (`flake = false` stays in profiles).

## Sequencing

1. Amendment (2026-08-19, owner ruling): no Go toolchain is added to dev
   VMs, and no rebuild gates Go development. On-VM Go work uses the tools
   repo devShell (`nix develop`, repo-pinned) or `nix shell nixpkgs#go`;
   Phase 1 was implemented and validated entirely this way. A dedicated
   go-dev VM profile is deferred until a need a shell cannot serve appears
   (editor tooling baked into a closure, or an egress-restricted VM that
   cannot reach the nix cache).
2. Within each tool phase: tools implementation PR merges → profiles + nexus
   cutover PRs merge → human runs flake update + rebuild → acceptance passes
   on a rebuilt VM → next phase.
3. Phase 4 deletion PRs merge only after all cutover PRs are merged and
   deployed, because writeShellApplication reads bash sources from the input
   at build time; deleting first would break the next profiles flake update.
4. Bug fixes to a tool mid-migration land in bash first, then mirror into the
   Go scenario matrix.

## Agent Gates

- (Removed by the 2026-08-19 amendment: no VM toolchain change exists, so
  no gate blocks on-VM Go work.)
- Every cutover requires a human flake update + `nixos-rebuild` on dev VMs
  and nexus. Blocks phase acceptance, which must run on the rebuilt VM.
- The agent cannot rebuild VMs, touch privacy VMs, or verify nexus-side
  wrappers; nexus verification is human (run `forge --help`, one read-only
  command, and `flake-status` on the host after rebuild).
- Live side-by-side `forge` comparison uses read-only commands only; no
  mutation commands against the real Forge during validation.
- `rotate-token`/rotation-suite migration is a deferred decision: requires a
  new issue and human sign-off; nothing in this plan touches it.

## Acceptance Tests

Phase 0 (any dev VM, no rebuild):

```sh
nix develop /home/allod/work/allod/tools -c go version   # repo-pinned toolchain
cd /home/allod/work/allod/tools
nix flake check
bash tests/allod-change.sh && bash tests/allod-patch.sh   # still green on bash defaults
for t in tests/forge/*.sh; do bash "$t"; done
```

Phases 1-3, per migrated tool (agent-runnable):

```sh
cd /home/allod/work/allod/tools
gofmt -l . | wc -l          # must print 0
go vet ./...
go test ./...               # includes httptest Forgejo fixtures for forge
nix build .#<tool>          # stdlib-only build, vendorHash = null
# dual-target parity where suites are exec-mocked:
ALLOD_UNDER_TEST="$(nix build .#allod --print-out-paths)/bin/allod" bash tests/allod-change.sh
ALLOD_UNDER_TEST=... bash tests/allod-patch.sh
CASCADE_UNDER_TEST=... bash tests/flake/flake-update-cascade/*.sh
# forge live read-only diff (parity evidence, run from a repo checkout):
diff <(forge pr list) <(result/bin/forge pr list)
diff <(forge issue list) <(result/bin/forge issue list)
```

Post-cutover, on a rebuilt VM (agent-runnable; also validates generated
lifecycle artifacts, not just source):

```sh
readlink -f "$(command -v forge)"   # points at the Go package, not writeShellApplication
forge pr list; allod change begin -h; flake-status >/dev/null
allod change begin -d smoke-test /home/allod/work/allod/tools   # protected repo → worktree path
allod change cleanup <that-path>
# token safety: the API path spawns no subprocesses, so no argv can carry the
# token; enforced statically plus a Go test pinning in-process-only auth
! grep -rn '"os/exec"' cmd/forge/ internal/forgeapi/
go test ./internal/forgeapi/ -run TestAuthHeaderInProcessOnly -v
```

Phase 4:

```sh
cd /home/allod/work/allod/tools
! test -e forge.bak && ! grep -rn "tools/forge\b" ../profiles ../nexus 2>/dev/null
go test ./... && for t in tests/workspace/*.sh tests/git-hooks/*.sh; do bash "$t"; done
```

## Rollback Plan

- Before a cutover merges: abandon the branch; nothing deployed changed.
- After a cutover, before Phase 4: revert the profiles/nexus wrapper commit
  (bash sources are still in the input) and rebuild; or pin the profiles
  `allod-tools` flake input back to the last-good rev recorded in the cutover
  PR body. Both are single-commit reverts plus a human rebuild.
- After Phase 4: deletions are plain git reverts in `allod/tools`; restoring
  a bash tool additionally requires re-adding its wrapper in profiles/nexus
  and rebuilding.
- Partial-phase state (tools PR merged, cutover not merged) is safe: Go code
  in the repo is inert until a wrapper references it.
- If a deployed Go tool misbehaves between rebuilds, the bash source remains
  executable directly from the checkout
  (`~/work/allod/tools/forge`, needs `jq`+`curl` which stay on dev VMs) as an
  immediate operator workaround.
