# microvm.nix Framework Adoption Review Prompt

You are the virtualization engineer people call when a NixOS module evaluates
perfectly, systemd reports green, and the guest still booted without its identity.
You read QEMU `fw_cfg` command lines, initrd ordering, mount namespaces, and
rotation state machines as one continuous execution trace. You know that
`User=microvm` is not tenant isolation, that `Requires=` is not a restart hook,
and that a writable image stops being disposable the moment it holds the only
copy of real work. This plan crosses all of those boundaries at once while
promising that libvirt stays boring. That is exactly your kind of problem. Do
not hold back.

## Your Task

Review the [microvm.nix framework adoption dev plan](../dev-plans/microvm-framework-adoption.md)
for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag
anything that will block implementation, create unnecessary work, or leave a
landmine for future changes.

Read the actual codebase and the pinned upstream microvm.nix source to ground
the review in reality. Do not review the plan in isolation. Trace the generated
QEMU runner, host and guest units, initrd and system credential flow, volume
mounts, Home Manager output, and rotation failure paths. Source-level
assertions are not enough for this change.

## Project Context

**Allod** is a self-sovereign NixOS stack in which one human-owned hypervisor
provisions isolated coding and privacy VMs from public framework code plus
private inventory, identity, and deployment data.

Key repos in play:

- `allod/strategy` - owns this public cross-repo plan and tracking issue
  `allod/strategy#20`. The final `allod/archetypes` integration PR is the sole
  PR that closes the strategy issue.
- `allod/vm` - owns shared guest behavior. It currently exports only
  `nixosModules.qemuGuest`, which imports disko, `disk.nix`, and
  `modules/qemu-guest.nix` as one unit.
- `allod/inventory` - owns machine facts and derives the non-hypervisor
  `lib.vmSpecsJson` projection plus committed `scripts/vm-specs.json`.
- `allod/archetypes` - composes inventory, identity, profile definitions, and
  guest builders; exports `vmFacts`, `nixosConfigurations`, and validation
  checks.
- `allod/profiles` - owns value-free public profile definitions. Its example
  dev Home Manager module currently hard-codes the Forge SSH key under
  `~/.ssh`.
- `allod/nexus` - owns the host NixOS module, provisioning scripts, and
  host-key and Forge-key rotation tools. Issues `allod/nexus#21`, `#22`, and
  `#23` are required parts of this arc.
- `microvm-nix/microvm.nix` - the proposed upstream runtime. The plan's spike
  uses revision `39a499ab85311b56dddb09ec43351cc3658f22c1` with nixpkgs
  `b6018f87da91d19d0ab4cf979885689b469cdd41`, which is also the nixpkgs
  revision currently pinned by `allod/vm`.

Current state (verify against the tree before trusting it; line numbers and
file layout drift):

- `allod/vm/flake.nix` exports one disk-installed guest module:
  `disko.nixosModules.disko`, `disk.nix`, and `modules/qemu-guest.nix`.
  `disk.nix` declares a GPT `/dev/vda` with `/boot` and `/`; the guest module
  imports nixpkgs's QEMU profile, enables systemd-boot, NetworkManager, and
  OpenSSH, and recursively fixes home ownership during activation. A microvm
  path cannot merely add an upstream module to this list.
- Inventory machines currently have `platform`, `type`, sizing, `ip`, `mac`,
  `forge_key`, and repo facts but no runtime fact. `vmSpecsJson` filters out
  hypervisors and projects a selected field set into JSON; the committed JSON
  is checked against it.
- Archetypes' `sharedModules` unconditionally imports
  `vm.nixosModules.qemuGuest` and agenix for both dev and privacy guests. It
  declares `/etc/ssh/<name>` as both the OpenSSH host key and age identity.
  `mkDevVm` then adds agenix consumers and an activation converter:
  `/root/.git-credentials`, `/etc/nix/netrc`, `/root/.netrc`,
  `/home/<user>/.netrc`, the Forge API token under the user's home, Forge SSH
  identity under `~/.ssh`, and registry-driven GitHub credential targets.
  Removing guest-local age therefore requires a complete runtime-specific
  consumer audit, not only deleting `age.identityPaths`.
- `archetypes/nix/vm-facts.nix` currently exports `ip`, `username`,
  `forgeKey`, `hostKeys`, and `hostKeySecretFile` for non-hypervisors. It
  fails loudly on missing required facts but knows nothing about runtime.
  Existing coherence checks compare these outputs with inventory and secrets.
- The Nexus host module currently enables libvirtd and installs host-side
  provisioning tools; it has no microvm.nix input or host module. The shared
  rotation library already contains deploy-flake `vmFacts` readers, but
  `vm-ssh-host-key` and `forge-ssh-key` still resolve target state from the
  working inventory/secrets checkouts and install activated keys directly into
  `/etc/ssh/<vm>` or `~/.ssh/<forge_key>` inside the guest.
- At the pinned microvm.nix revision, `microvm.credentialFiles` is an
  `attrsOf path` and QEMU renders each entry as
  `-fw_cfg name=opt/io.systemd.credentials/<name>,file=<host-path>`.
  Only the QEMU runner implements this transport. The x86_64 default QEMU
  machine is `microvm`; the guest root defaults to tmpfs; `storeOnDisk`
  defaults true when `/nix/store` is not shared.
- The upstream host module creates one `microvm` system user in group `kvm`
  and runs every `microvm@<name>` instance as that same principal from
  `/var/lib/microvms/<name>/current/bin/microvm-run`. The template unit has
  `Restart=always` and depends on TAP, device, virtiofsd, and booted-link
  helpers. It does not provide per-instance filesystem isolation. Its state
  directory is `microvm:kvm` mode `0775`, and `tap-up` hardcodes
  `user = "microvm"`.
- That same import makes four assignments outside `microvm.*`:
  `hardware.ksm.enable = lib.mkDefault true` where the NixOS default is
  `false`; upstream's `microvm` command on every account, whose `-r` execs the
  runner with no unit and whose `-s` execs `ssh` with
  `StrictHostKeyChecking=no` into a root shell over VSOCK; a
  `boot.kernelModules` list merge; and a `security.pam.loginLimits` `memlock`
  grant that never applies because the generated units set no `PAMName`.
- At kernel 6.12.93, `/proc/<pid>/root` opens under `PTRACE_MODE_READ_FSCREDS`
  and is gated only by the same-credential comparison in
  `__ptrace_may_access`. Yama's hook inspects `PTRACE_MODE_ATTACH` only, and
  every `has_pid_permissions` branch reduces to that same read check, so no
  `ptrace_scope`, `hidepid`, or `subset=pid` value closes a same-uid read into
  another process's mount namespace. Measured downstream and re-derived from
  the pinned kernel source; do not relitigate it.
- Upstream `microvm.volumes` defaults `autoCreate = true`; creation touches,
  truncates, and formats the declared host image if it does not exist.
  Volumes are attached after the read-only store disk and are mounted by label
  or virtio block letter. `microvm.shares` defaults empty, but the upstream
  virtiofsd template still exists and is skipped by `ConditionPathExists` when
  its runner script is absent.
- The public/private boundary is load-bearing. Public repos may carry
  synthetic, buildable examples and value-free option shapes, but real
  credential sources, image paths and sizes, addresses, routes, TAP
  attachment, and cutover actions belong to private deployment composition.
  Agents run inside dev VMs; real host rebuild, key handling, and cutover are
  human-only.

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections
from `dev-plans.md`: Tracking Issue, Goal, Scope, Risk Assessment, Interface
Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Agent Gates apply
here and may not be omitted.

This is a multi-repo, multi-PR plan. Verify that each PR or milestone has a
defensible residual risk, prerequisite PRs use `Refs allod/strategy#20`, Nexus
issue PRs close their own issues, and only the final Archetypes integration PR
uses `Closes allod/strategy#20`. Confirm that every cross-repo contract is
available before the next consumer PR needs it and that rollback unwinds pins
in the reverse dependency order.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have
problems. These are lenses, not checklists - follow the thread wherever it
leads.

The six standing lenses in `dev-plans.md` (internal consistency, operational
sequencing, risk calibration, acceptance-test coverage, rollback fidelity,
generated lifecycle behavior) apply as defaults on top of the plan-specific
areas below.

Pass 5 verified and closed the three previous focus areas: the final root
`ExecStopPost` merged with `lib.mkAfter` lands after the upstream unregister
hook and systemd will not start the unit again until it completes, the
graceful-stop sentinel plus broken-socket sabotage does distinguish a QMP
shutdown from a systemd kill, and the QMP directory topology is right. Do not
reopen them. The isolation claim they rested on was wrong, and pass 5 replaced
the shared runner uid with one principal per VM. That replacement is now the
riskiest text in the plan.

1. **Per-VM runner principal against the pinned upstream.** Scope the review to
   pass 5's commits `e315741`, `86dd6e5`, `d40adf9`, `a7e2fca`, and `e95d704`,
   not a full re-review. Contracts 16, 16a, and 16b now require a
   `microvm-<name>` user per selected guest. Prove that is implementable
   without breaking what the shared uid made work: `tap-up` hardcodes
   `user = "microvm"`, `install-microvm-<name>` chowns the state directory and
   `current` to `microvm:kvm`, `microvm-set-booted@` runs as `microvm:kvm` and
   writes `booted`, `/dev/kvm` access comes from group `kvm`, and the host
   tmpfiles rule the plan now tightens is upstream's. Confirm nothing else
   still depends on one shared principal, that the per-VM ownership of volume
   images is a change the private deployer can actually make, and that
   collapsing both guests onto one principal is a sabotage fixture that can be
   built. Check that dropping the `kernel.yama.ptrace_scope` pin removed no
   control the rest of the plan relies on.

2. **Non-pageable plaintext roots and their mount ordering.** Contract 8a makes
   the host module mount `ramfs` or `noswap` `tmpfs` at the host credential and
   rollback roots. Prove the mount is expressible in the host module and lands
   before every writer: the launcher on each start, the rotation tools which
   run outside any unit, and anything that creates the roots through tmpfiles.
   Check `noswap` availability at the pinned nixpkgs and kernel, whether an
   unbounded `ramfs` is a real host hazard here, whether the rollback slot
   survives across the mount's own lifecycle and a host reboot as contract 19
   still claims, and whether the stock-`/run` sabotage fixture can fail for the
   right reason.

3. **Cross-repo agreement on the root options.** The three plaintext roots are
   now options rather than literals, while contract 7 still requires each
   `microvm.credentialFiles` value to be an absolute string under the
   per-machine runtime directory. Those two live in different repos. Prove the
   guest's declared credential paths derive from the host module's option
   instead of a second copy of the literal, that the host/guest agreement check
   in contract 17 compares derived values, and that the option-change assertion
   in the generated-artifact list is executable rather than aspirational.

Next pass: scoped diff review of the five pass-5 commits above, not a full
pass. Use a model other than `claude-fable-5`, which authored them, and other
than `gpt-5.6-terra`, whose QMP/lifecycle fixes regressed immediately more than
once. `gpt-5.6-sol` is eligible again: it authored `23e9704`, which pass 5
verified as stable, and it is the roster's default for cross-repo generated
lifecycle behavior. Confirm the model is instantiable on its runner before
recording it — pass 4 recommended `claude-opus-4-6`, which the Claude runner
cannot instantiate, and pass 5 substituted `claude-fable-5`.

## Pass Metadata

Pass 1 found three original-plan BLOCKERs, one original-plan GAP, one
original-plan SIMPLIFY, and one GAP introduced by `416bd94`. Commit `416bd94`
fixed the pin handoff, disabled automatic image creation, and removed the
generic volume manager, but its launch and rollback designs required
blocker-level repair in pass 2; it remains unstable. Commit `12c49d7` resolved
the raw/wrapped name collision but added an unused public raw guest export that
pass 2 removed, so it also is not fix-stable.

Pass 2 found two BLOCKERs and one SIMPLIFY introduced by the prior fixes, plus
one original-plan GAP missed by pass 1. Commit `10f3425` makes the root helper
the sole privilege transition, adds a recovery selector that overrides the
still-staged source, makes required volume mounts fail in the initrd, and
removes the raw guest export; its stability is pending the scoped next pass.
The SIMPLIFY sweep reconsidered the raw guest export, root launcher, rollback
slot, volume creation/capacity, and a generic mount validator. It removed only
the unused raw export because the other mechanisms serve concrete lifecycle or
security requirements. Convergence has not been reached: this is the first
pass where review-introduced findings outnumbered original-plan findings, and
the pass contains both BLOCKERs and an original-plan GAP.

Pass 3 found one BLOCKER introduced by `10f3425`: the new launch namespace did
not make QEMU's `microvm.socket` visible to the upstream host-namespace
`ExecStop`, so graceful stop and restart could fall through to a systemd kill.
Commit `e99f5e6` requires a root-owned, `microvm:kvm`-writable, per-VM QMP
working directory bound at the same absolute path inside the launcher and adds
generated plus nested coverage for stop, retry, rebuild, and rollback. The
SIMPLIFY sweep reconsidered a separate shutdown helper, a generic host-state
manager, a second service unit, and a durable QMP path; all add scope or weaken
the one-start lifecycle, so nothing was removed. Commit `10f3425` is unstable;
`e99f5e6` is pending an independent scoped diff review. The review-introduced
finding majority now spans two passes, which meets the plan-text convergence
heuristic, but convergence is not declared until the mandatory independent
review of the new structural fix verifies that the terminal lifecycle repair
does not introduce another blocker. Afterwards, hand remaining work to
implementation review unless that scoped diff exposes a new plan defect.

Pass 4 found one BLOCKER and two GAPs, all introduced by `e99f5e6`.
First, the execing root launcher could not perform cleanup after QEMU exit and
upstream post-stop hooks; `23e9704` assigns idempotent cleanup to a final root
`ExecStopPost` merged with `lib.mkAfter` and covers preparation failure, QEMU
failure, every stop/restart path, and rollback-slot preservation
([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).
Second, cleanup plus restart could falsely pass as QMP shutdown because the
pinned shutdown script accepts an absent socket before systemd kills QEMU;
`23e9704` requires a durable guest shutdown sentinel and a broken-socket
sabotage fixture. Third, the same-uid fixture omitted the new writable QMP
socket/directory, current runner entries, and their host source parents;
`23e9704` adds those objects and operations to every namespace access path
([allod/nexus#23](https://forge.anarch.diy/allod/nexus/issues/23)).
The SIMPLIFY sweep considered a cleanup service, a long-lived privileged
supervisor, a generic runtime-directory manager, and a second shutdown helper;
all were rejected in favor of one ordered post-stop command on the existing
unit. Commit `e99f5e6` is unstable. Commit `23e9704` is pending independent
scoped verification. The review-introduced finding majority continues for a
third pass, but convergence is not declared while this replacement
blocker-level lifecycle fix lacks its mandatory independent stability result.
If the scoped next pass finds no new blocker or original-plan GAP, stop
plan-text review and hand the remaining checks to implementation review.

Pass 5 was the scoped diff review of `23e9704`, widened by the user to also
dispose of three downstream integration reviews left unaddressed on
`allod/strategy` PR #22. `claude-fable-5` ran it: pass 4 recommended
`claude-opus-4-6`, which the Claude runner cannot instantiate, so the
substitute was chosen as the highest-capability Claude option, the roster's
role for terminal verification of cross-repo generated lifecycle behavior, and
not the author of any fix in this plan. Record substitutions like this one
rather than silently renaming the pass.

Two determinations, kept separate. First, `23e9704` is fix-stable: its ordered
final `ExecStopPost`, its `lib.mkAfter` merge, its stop-before-next-start
guarantee, and its sentinel-based graceful-stop evidence all survived scoped
verification against the pinned unit, the built runner argv, and the built
`microvm-shutdown`. Only its own new "refuses a stale socket" wording was
wrong, and that is a GAP, not a blocker. `gpt-5.6-sol` is therefore the first
model on this plan with a stable structural fix. Second, the plan text has not
converged: pass 5 found three BLOCKERs and four GAPs, five of them original
defects that four passes missed, so both stop conditions fail. The
review-introduced majority streak is broken, and this pass contains both new
BLOCKERs and original-plan GAPs.

Those five original defects share a cause worth naming: every earlier pass
reviewed the plan's own mechanisms and never audited what the pinned upstream
host module does to the host, nor whether a control the plan names actually
holds at the kernel. A sweep that asks only "did this change a pre-existing
default?" sees the overrides and misses the additions. The plan's isolation
contract asserted a boundary that measurement showed open, and its
"ramfs-only" title had no mount behind it.

Pass 5 replaced the shared runner uid with one principal per VM, put both host
plaintext roots on declared non-pageable storage behind readable options,
bounded the host-wide settings the upstream import brings, and made the
leftover-QMP-directory behavior explicit. That is a structural change, so pass
6 is its scoped diff review by a different model.

Do not re-open focus areas addressed in previous passes unless the current
plan contradicts itself.

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves.
  Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. Public option
  and module interfaces may break, but the plan explicitly promises that
  selected libvirt guests keep working throughout this migration.
- **Do not overengineer.** If the plan introduces abstraction that is not
  needed for the concrete public examples and one private deployment, call it
  out. Three explicit runtime branches beat a premature framework. Run an
  explicit SIMPLIFY sweep every pass: actively hunt for scope, ceremony, or
  abstraction to delete rather than treating a quiet pass as nothing to cut.
- **Solo project, one human.** No team coordination overhead. No release
  process. No migration guides for other consumers.
- **Security matters, ceremony does not.** Host/guest credential isolation,
  no durable plaintext, stable SSH identity, fail-closed startup, and
  same-principal sibling isolation are real boundaries. Extra schemas and
  generalized managers are not.
- **Solve problems as they come.** If the plan front-loads snapshotting,
  scheduling, backup orchestration, multi-host behavior, or generic migration
  state, remove it.
- **Think operationally.** Walk cold boot, rebuild, restart, crash, rotation,
  missing credential, failed mount, and rollback in the order systemd and the
  scripts actually execute them.
- **Calibrate residual risk.** The public implementation remains R3 if
  synthetic nested and generated-artifact validation contains its worst
  failures. Challenge any milestone whose validation or rollback leaves
  unique volumes, credential authority, or cross-repo pins uncertain.
- **Inspect generated lifecycle artifacts.** Do not stop at Nix evaluation.
  Read the actual QEMU argv, unit dependency graph, service credentials,
  activation scripts, Home Manager files, mounts, closures, and failure logs.

The person implementing this is technically sharp. They do not need
hand-holding; they need the sharp edges they missed.

## Severity Rubric

Use `[BLOCKER]` only when following the plan literally is likely to:

- perform a destructive or unsafe operation;
- fail before implementation can complete;
- leave the resulting system nonfunctional;
- break first boot, activation, provisioning, rebuild, rotation, or rollback
  lifecycle behavior;
- violate a security or privacy boundary; or
- require missing human input that cannot be inferred from the repo or memory.

Use `[GAP]` for missing or contradictory plan details that could cause rework,
test blind spots, stale docs, or implementation ambiguity, but where a
competent agent with workspace memory could still proceed safely.

Use `[GAP]` when the Risk Assessment is missing, materially understated,
materially overstated, or unsupported by the acceptance tests and rollback
plan. High residual risk is not a blocker by itself; it should drive better
validation, clearer rollback, or a human gate only when a real human-only
action or decision exists.

Use `[SIMPLIFY]` for unnecessary scope, ceremony, or abstraction. Commit
SIMPLIFY fixes when they remove implementation work, delete unnecessary scope,
or prevent an unnecessary abstraction. Do not create plan commits for
wording-only simplifications unless the wording changes execution behavior.

Use `[QUESTION]` only when the plan cannot be corrected from repo context. If
the answer is inferable, resolve it as `[GAP]` or `[SIMPLIFY]`.

Do not classify duplicated workspace policy, phrasing improvements, or
reminders already covered by memory as findings unless the plan directly
contradicts that policy.

## Deliverable

The deliverable is not a report. Every review pass ends with:

1. Plan-file commits for findings that require plan changes, or an explicit
   no-findings result.
2. A final review-prompt commit updating this prompt's Focus Areas and pass
   metadata.
3. A push to the remote.

For each finding that requires a plan change, edit the plan and commit the fix.
Group changes into logical, self-contained commits.

A one-line commit is fine when it records a real implementation decision. Fold
or skip commits that only rephrase already-correct guidance.

## Output Format

Give your review as a numbered list of findings, each tagged as one of:
`[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, `[QUESTION]`. Start with blockers, end with
questions. Be blunt.

If a design decision is sound, say so briefly - do not damn with faint praise.
If something is right, name it and explain why so a later pass does not undo
it. QUESTIONs must be resolved in the plan, not left as open items. If the
answer is clear from the codebase, update the plan and commit. If the answer
requires human input, add the question to the Focus Areas section for the next
pass.

## After Each Pass

As a final commit, update this prompt's Focus Areas section:

1. Remove focus areas that were fully addressed.
2. Refine any focus areas that were partially addressed.
3. Add new focus areas discovered during the review.

The focus areas should always reflect the most productive targets for the next
review pass, not a historical record of past ones.

Include a plain-text findings summary in this commit's message:

- Count only findings new to this pass, by tag. Carried-over unresolved items
  are not findings; move them to Focus Areas rather than re-counting them, so
  the per-pass counts track a real severity trend instead of inflating it.
- Give each new finding a numbered entry with its tag, short title,
  one-sentence explanation, fixing commit hash, and issue link if one exists.
- Classify each finding's origin: an original-plan defect, or introduced by an
  earlier review pass (name the commit that introduced it). Origin is what
  makes the convergence heuristic below and any trend read possible.
- State what the SIMPLIFY sweep considered for deletion this pass, even when
  nothing was cut. Two or more consecutive passes with zero SIMPLIFY on a
  growing plan is a smell to call out, not a clean bill: pure accretion is what
  breeds internal contradictions.
- Put a final `Model: <exact model>` footer at the bottom (for example,
  `Model: claude-fable-5` or `Model: gpt-5.5`). Use the exact identifier of the
  model that actually ran, not the agent framework, the product name, or the
  model a previous pass recommended. This is review-pass metadata, not
  authorship attribution.

When a pass commits a structural or design change (a blocker-level fix), the
next pass should be a scoped diff review of that change, not a full re-review,
run by a different model than the one that wrote the fix. Structural fixes are
where new blockers enter, and the author model tends not to catch its own
gaps. Passes are launched manually one at a time, so you cannot pick or start
the next model yourself; make the handoff explicit instead. In the Focus Areas
update, add a `Next pass:` line that names the commit(s) to review, says
whether it is a scoped diff or a full pass, and recommends a model other than
the fix's author (preferably the most fix-stable one on record; see Agent
Rotation in `dev-plans.md`). Whoever starts the next pass reads that line and
the previous `Model:` footer and picks accordingly. Record in the same update
how the fix held up so its stability stays traceable.

Stop the plan-text review when either condition holds:

- Review-introduced findings outnumber original-plan findings for two
  consecutive passes.
- Two consecutive passes produce no BLOCKER and no original-plan GAP.

At that point the plan text has converged; hand any remaining focus areas to
implementation review, and resolve remaining SIMPLIFYs during implementation.
The old zero-BLOCKER/zero-GAP/zero-QUESTION rule could grind a token budget to
nothing without terminating, because each accretive fix tended to seed the
next finding.

## Before Final Response

- Plan fixes are committed, or the pass explicitly found no plan changes.
- This review prompt's Focus Areas are updated and committed.
- The final review-prompt commit message includes the findings summary and
  `Model:` footer.
- The repo is pushed to the remote.
- `git status` is clean.
