# microVM Runtime Credential Delivery Review Prompt

You have spent a career on the exact seam where a secret stops being ciphertext and starts being a file on a running machine. You know what `LoadCredential=` does when the credential is not there, you know which of `/run/credentials/@system` and `$CREDENTIALS_DIRECTORY` a unit can actually open, and you have personally been burned by a host key that regenerated itself at 3am and turned an anti-TOFU pin into a wall. You read NixOS module merge order the way other people read a shopping list, and you have a specific grudge against a security control whose only proof is that someone wrote it down.

This plan moves a VM's private key material off an encrypted-at-rest, decrypted-in-guest path onto a per-boot, host-memory path. It is the highest-risk change in its arc. Go break it. Do not be polite, and do not grade on effort.

## Your Task

Review [microvm-credential-delivery.md](../dev-plans/microvm-credential-delivery.md) for gaps, wrong premises, and landmines. Be direct and specific. Flag anything that blocks implementation, creates unnecessary work, or leaves a trap for the slices that follow.

**Read the actual code and the actual upstream sources.** This plan makes many empirical claims, and two passes have now shown that its *newest* text is where the wrong ones live. Pass 2 re-derived everything pass 1's fix commit added and found one claim wrong — nixpkgs defines `systemd.services."sshd@"` unconditionally inside `mkIf cfg.enable`, not under `mkIf cfg.startWhenNeeded` — while the rest held, as did the older claims pass 1 had already re-derived.

The claims added by pass 2's fold (`8bab682`) have been checked by nobody:

- NixOS renders `wantedBy`/`requiredBy` as `.wants`/`.requires` **symlinks**, reachable at `config.environment.etc."systemd/system".source` and never as a line in either unit's fragment or in the target's.
- Nix 2.31.5 runs the `git` binary as a subprocess for `git+https` network operations, so those fetches resolve credentials through git's own helper chain rather than through a netrc. The plan states this as measured with a PATH shim.
- `pkgs.formats.nixConf`'s `checkPhase` runs `nix config show` and promotes an unknown-setting *warning* into a build failure, which is why the preservation control must be a real setting.
- Pinned Home Manager writes `home.file.".ssh/config".text`, renders `IdentityFile` unquoted one line per list element, and types `identityFile` as `either (listOf str) (nullOr str)` — so two plain-priority string definitions conflict rather than concatenate.
- `modules/github-credentials.nix` derives its own targets from the flake's `secrets` input, with no route in from `mkDevVm`, which is why the plan now threads one.

Verify against the tree and the pinned store paths. **A wrong empirical claim is the highest-value finding you can return.**

Pinned sources, all readable locally:

- microvm.nix `39a499ab85311b56dddb09ec43351cc3658f22c1` at `/nix/store/yyrsgw6gs5v1qnfn839mrn6mmj91zwfw-source` (narHash verified against `allod/vm` `flake.lock`).
- nixpkgs `b6018f87da91d19d0ab4cf979885689b469cdd41` at `/nix/store/4k79ns9drp02wyvcix81c7nnz4hn8psi-source` (narHash verified).
- agenix at `/nix/store/kg2mkqk4qkw5aqv1569cs5szhr71bc8y-source`.
- systemd 258.7 source at `/nix/store/a08jd53b59v5izippz0l5jpd61amra26-source`.
- QEMU 10.1.5 at `/nix/store/k8zfisphlb6sddc03r8p76xyga3s77wj-qemu-10.1.5` (source tarball `/nix/store/5cp2l2c344w5qf8b31fqnysscg1l2ylc-qemu-10.1.5.tar.xz`).
- Home Manager `3ee51fbdac8c8bdfe1e7e1fcaba6520a563f394f` at `/nix/store/y9jlpk6lrhf7lhnj9b486fzdps88pdid-source` (narHash verified against `allod/vm` `flake.lock`; `archetypes` follows `vm/home-manager`). A second Home Manager source is in the store from the agenix input — check the narHash before reading `modules/programs/git.nix`.
- git 2.51.2 documentation at `/nix/store/rsk6pv03hs9jji2v508ci816m9d7jshn-git-2.51.2-doc`; `share/doc/git/gitcredentials.adoc` is where the helper-list semantics are.

If a store path has been garbage-collected, find the current one rather than assuming the claim; say so if you cannot.

## Project Context

**Allod** is a self-sovereign NixOS VM stack for agentic coding and privacy tasks.

Repos in play, local checkouts under `/home/allod/work/allod/`, all at origin tip:

- `archetypes` — the VM framework; one `flake.nix` builds every machine. **This is the repo being changed.**
- `vm` — guest and host NixOS modules; owns `nixosModules.qemuGuest` / `nixosModules.microvmGuest` and the sole microvm.nix pin. Not changed here.
- `nexus` — the hypervisor host. Its microVM host module and root launcher are **already merged** and own everything host-side.
- `secrets` — identity and credential registries, synthetic in public.
- `inventory` — machine data. No public machine declares `runtime = "microvm"`.
- `profiles` — public example home configurations. Deliberately not changed; see the plan's Scope.

Current state, verify before trusting:

- `archetypes/flake.nix` is over 2000 lines and grows with every slice, so **locate constructs by identifier, not by line number** — three citations in the plan's first draft had already drifted onto unrelated code by pass 1. `sharedModules` sets `services.openssh.hostKeys` and `age.identityPaths` to `/etc/ssh/<name>` for every guest — the SSH host key and the age identity are the same file.
- `microvmVolumesModule` is the direct precedent: gated with `lib.optional (runtime == "microvm")` on the builder's module list, a typed option with a reused `pathErrors` validator, assertions over merged config. Its consuming check `runtime-module-selection` has 16 sabotage fixtures, pins diagnostics through `missingDiagnostic` against `config.assertions`, and probes the libvirt boundary with `lib.hasAttrByPath`.
- That check states its own memory constraint in the comment above its `sabotages` list: fixtures are consumed as thunks because holding them live is a measured OOM kill, and they take `nix flake check` from ~4 GB to over 5 on a 7 GiB box with **no swap**.
- Agents run in dev VMs. No host access, no real credentials, no boot.

Parent plan: [microvm-framework-adoption.md](../dev-plans/microvm-framework-adoption.md), converged at review pass 10. This slice implements contract **7**, the guest half of **8a**, and **9, 10, 11, 12, 14**. Do not re-litigate the parent's settled contracts — but if this plan or the code **contradicts** one, that is a finding, and **record it in this plan's "Contract contradictions found" section rather than editing the parent**. The plan records four such contradictions. Pass 2 checked the first three and named each a right call. The fourth is pass 2's own, added in `8bab682` and reviewed by nobody: the two-unit split deliberately breaks parent contract 10's last sentence for Forge credentials, because a broken Forge credential fails its materializer and, by design, no consumer unit. Check that call, and check whether there are more.

## Structural Conformance

Verify every section `dev-plans.md` requires: Tracking Issue, Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates, Acceptance Tests, Rollback Plan. This is one PR, so per-PR risk does not apply. Verify the closing keyword: `Closes allod/archetypes#29`, and it must **not** close `allod/strategy#20`.

## Focus Areas

The six standing lenses in `dev-plans.md` apply as defaults. Concentrate here.

Two passes have run and most of what this section used to carry is closed. Pass 1's four blockers changed the design; pass 2 verified that design end to end and found no blocker — the two-unit split, its dependency direction, the `Before=sysinit.target` coverage, the reactivation ordering, the two-file Git rewiring and the `mkAfter` appends all survive. What is left below is the residue of those checks plus everything pass 2's fold introduced.

The record now shows the pattern to work with: pass 2's eleven findings split six review-introduced to five original-plan, so **the newest text is the least reviewed text**. Read `8bab682` adversarially, and do not treat a claim as verified because it is written as measured.

1. **The pull symlinks, new in `8bab682`, and the only thing asserting either materializer ever runs.** Test 13 used to read the reactivation edge off `sysinit-reactivation.target`'s rendered unit, which never carries it, and nothing at all asserted the Forge unit's `wantedBy = [ "sysinit.target" ]` — its only start path, since nothing requires it. The test now asserts four symlinks under `config.environment.etc."systemd/system".source`, paired with a sabotage that `mkForce`s the Forge unit's `wantedBy` empty and must lose exactly one needle while every fragment needle stays green. Verify that artifact is reachable from a check, that the four names are spelled the way `generateUnits` emits them, and that the sabotage fails for its named reason rather than by breaking evaluation.

2. **Two new threading routes, both changing a builder signature.** `mkDevVm` gains `githubCredentialTargets ? secrets.lib.githubCredentialTargets.${name} or []`, threaded through `mkGithubCredentialModule` into `modules/github-credentials.nix` as a `targets` argument replacing its internal lookup — a second edited module, which Scope and the Rollback Plan now both state. Verify it is genuinely drvPath-neutral, that nothing else calling that module or reading its output changes (`credential-profiles`, the forgejo-token target checks), and that an injected target really reaches the microvm refusal. Separately, test 2's three fixtures now build from `runtime-module-selection`'s `devFixture` rather than `dev-forge-opt-out`'s `withTokens`: check that `devFixture`'s `// args` carries an overridden `identity` and a `forgeKey` through to `mkDevVm`, and that `dev-forge-opt-out` genuinely needs no change.

3. **The `[credential]` header on the appended `/etc/gitconfig` block, and an assertion the plan admits has no fixture.** Bare `helper =` lines appended to a `types.lines` option belong to whatever section precedes them once the definitions are concatenated. The block now opens its own `[credential]` header and test 19 asserts it *by section* — the nearest preceding `[…]` must be `[credential]` — while recording that reaching that failure needs a third `gitconfig` definition and therefore another forced dev system, which the memory budget will not buy. Is that the right call, or is there a cheaper sabotage? And does a repeated `[credential]` section behave as claimed in git 2.51.2?

4. **The netrc rewrite rests on one measured mechanism, and it is the highest-value claim in the fold to attack.** The consumer column is now "the Nix daemon and any nix client's curl-based fetches, via nix.conf `netrc-file`", and "one netrc, not three" now rests on Nix 2.31.5 shelling out to the `git` binary for `git+https` network operations, expressly superseding the libgit2 rationale. If Nix can reach the network for a `git+https` input by any path that does not run `git`, a microvm guest loses credentials on flake fetches and the justification for dropping the user netrc goes with it. Note what the plan does **not** claim: it does not say `~/.netrc` has stopped working, only that a *user* netrc is unnecessary here. Do not review it as the stronger claim.

5. **The consumer table's modes against their real consumers. Neither pass has measured this half.** Pass 2 confirmed the `0640 root:users` pair is not a new disclosure — `modules/netrc.nix` already gives the dev user the same bytes — and corrected who reads the netrc, but stopped there. Still unmeasured: will sshd accept a host key at `0600 root:root` under `/run`, will `ssh` accept an `IdentityFile` at `0600 <user>:users`, will an unprivileged `nix` client actually open a `0640 root:users` netrc, and does anything need the *public* half of a key this table does not deliver?

6. **Can the checks fail — two new sabotages, and the same class swept again.** Pass 2 found the persistent-mount scanner (test 23) with no paired sabotage at all, and test 20 reading an option in a section whose first sentence bans option reads. Both changed in `8bab682`: test 23 rides the unit-mutation fixture with a credential-copy line under the dev home mount, and test 20 reads the rendered `~/.ssh/config` with a needle that tolerates both realizable sabotage shapes. Verify each fails for its named reason, then sweep the list again for the same class — an assertion whose deletion leaves every fixture green.

7. **Memory, and this fold's net cost.** The check peaks over 5 GB on a 7 GiB box with no swap. `8bab682` merges test 5's moved-credential-root fixture into the moved-image-root fixture that was already forced (one dev system back) and adds two that cannot share a fixture: the `wantedBy = []` sabotage, which needs every other needle green, and test 23's copy mutation, which must evaluate far enough to render the artifacts the scanner reads. Net is one more forced dev system than before. Is that affordable, and is any surviving fixture still mergeable?

### Settled by pass 1 — do not undo without new evidence

`gpt-5.6-sol` examined these and named each as correct. Re-deriving them is wasted effort; *reversing* one needs a measurement, not an opinion.

- The absolute-path `LoadCredential=<name>:/run/credentials/@system/<name>` form is correctly fail-closed, and the bare form's soft-fail is the reason.
- `hostKeys = []` plus one explicit runtime `HostKey` plus a masked `sshd-keygen` correctly prevents regeneration.
- Keeping the agenix module imported with `age.secrets == {}` is sound — it keeps the option declared so checks can assert `== {}` rather than probe for absence.
- Same-directory temporary writes followed by `mv -T` are sound.
- Rejecting `systemd-tmpfiles` is justified, for the three reasons contract 6 records.
- R3 is correctly calibrated. It becomes R4 if this PR enables a deployed machine or is accepted as sufficient evidence for cutover.

### Settled by pass 2 — same rule

`claude-fable-5` measured each of these against the pinned sources and named it correct. It also spot-checked every citation in the plan and all resolved; they are now in identifier form, so a later pass checks the identifier, not a line.

- **The two-unit split and its dependency direction.** The host-key unit carries no `Condition*`, so it can fail but never be skipped, and every path that starts the Forge unit pulls it first — one root creator, nothing stranded.
- **Test 13 genuinely distinguishes a merged implementation from a split one**, and the no-attribute assertion on the privacy and no-Forge fixtures additionally forces the composed-outside-`mkIf` shape, since a `mkIf false` would still create the attribute.
- **`After=systemd-sysusers.service` covers all three user-creation paths**: `userborn.service` carries the alias, `systemd.sysusers.enable` keeps the upstream unit name, classic activation creates users in stage 2 before any unit, and a nonexistent unit in `After=` is a no-op.
- **`Before=sysinit.target` covers every claimed consumer, sockets included** — verified in systemd 258.7's `service_add_default_dependencies` and `socket_add_default_dependencies`.
- **Reactivation needs both `requiredBy` and `before`**; pinned `userborn.nix` carries the target in both lists.
- **The no-`mkForce` append pair for `nix.conf`**, including last-occurrence-wins measured on Nix 2.31.5 and a preservation control that can genuinely fail.
- **The empty-string helper reset works across git's config file order**, not merely within one file (`gitcredentials.adoc:199-202`).
- **The double-rendering trap is real**, with one nuance kept in the plan: `multipleType`'s `either` merge means two *list* definitions concatenate rather than conflict, so test 19's ordered-list assertion — not a merge error — is what pins the final helper sequence.
- **`hostKeys = []` as a lexical definition rather than an omission**, `sshd-keygen.enable = false` rendering a real `/dev/null` mask, and `HostKey` as a list under `settings` throwing.
- **The `0640 root:users` pair is not a new disclosure**, and `forgeKey ? machines.${name}.forge_key` read from `machines` rather than `vmFacts` is the right shape.
- **All recorded contract contradictions are the right calls**, including contradiction 3 re-verified end to end. The fourth, added in `8bab682`, is pass 2's own and has not been reviewed.

## Review Guidelines

- **Forward momentum is king.** No style nitpicks, no nice-to-haves. Only what will actually cause problems.
- **No backwards compatibility required.** Pre-alpha. Any interface can break.
- **Do not overengineer.** Three similar lines beat a premature helper. Hunt for scope and ceremony to delete every pass.
- **Solo project, one human.** No team coordination, no release process, no migration guides.
- **Security matters, ceremony does not.** The privacy and security boundaries must be airtight; everything else can be pragmatic. On this plan, that cuts both ways — call out a control that is only theatre as loudly as a missing one.
- **Think operationally.** What happens when someone executes this cold, or in the wrong order, or on the machine they are sitting in front of?
- **Inspect generated lifecycle artifacts.** Do not stop at source evaluation. Units, activation text, `sshd_config`, `nix.conf`, Home Manager output, closures.

## Severity Rubric

Use `[BLOCKER]` only when following the plan literally is likely to: perform a destructive or unsafe operation; fail before implementation can complete; leave the resulting system nonfunctional; break first boot, activation, provisioning, rebuild, rotation, or rollback lifecycle behavior; violate a security or privacy boundary; or require missing human input that cannot be inferred from the repo or memory.

Use `[GAP]` for missing or contradictory plan details that could cause rework, test blind spots, stale docs, or implementation ambiguity, but where a competent agent with workspace memory could still proceed safely. Also use `[GAP]` when the Risk Assessment is missing, materially understated, materially overstated, or unsupported by the acceptance tests and rollback plan.

Use `[SIMPLIFY]` for unnecessary scope, ceremony, or abstraction.

Use `[QUESTION]` only when the plan cannot be corrected from repo context. If the answer is inferable, resolve it as `[GAP]` or `[SIMPLIFY]`.

Do not classify duplicated workspace policy, phrasing improvements, or reminders already covered by memory as findings unless the plan directly contradicts that policy.

## Deliverable

The deliverable is not a report. Every review pass ends with:

1. Plan-file commits for findings that require plan changes, or an explicit no-findings result.
2. A final review-prompt commit updating this prompt's Focus Areas and the Review Evidence table below.
3. A push to the remote.

## Output Format

A numbered list of findings, each tagged `[BLOCKER]`, `[GAP]`, `[SIMPLIFY]` or `[QUESTION]`. Blockers first, questions last. Be blunt. If a design decision is sound, name it and say why, so a later pass does not undo it.

## After Each Pass

Update the Focus Areas above — remove what is closed, refine what is partial, add what you found — and append a row to the Review Evidence table. In the final commit message include a plain-text findings summary: count only findings **new to this pass** by tag; give each a numbered entry with tag, short title, one-sentence explanation and fixing commit; classify each finding's origin as an original-plan defect or introduced by a named earlier commit; state what the SIMPLIFY sweep considered even if nothing was cut; and end with a `Model: <exact model id>` footer.

Add a `Next pass:` line naming the commits to review, whether it is a scoped diff or a full pass, and a recommended model **other than** the one that authored the fix.

Stop the plan-text review when either holds:

- Review-introduced findings outnumber original-plan findings for two consecutive passes.
- Two consecutive passes produce no BLOCKER and no original-plan GAP.

## Review Evidence

**Two review passes have run.** Pass 1 found four blockers and changed the design; pass 2 found none, which is the first evidence that the design is right rather than merely written down. **The stopping rule has not started counting.** Pass 2 produced no blocker but did produce original-plan gaps, so it is not the first of the two clean passes, and review-introduced findings have outnumbered original-plan ones for one pass only.

Commit provenance, which is what the origin column below refers to:

| Commit | What it is |
|---|---|
| `7ec057a` | The original plan and this prompt. |
| `9e82896` | Author self-correction: four inferences replaced with measured behaviour from pinned nixpkgs, systemd and OpenSSH, and ordering moved to `Before=sysinit.target`. |
| `94c5ad6` | Author self-correction interrupted by a session limit. Its contract 1 and 3 edits were kept; its half-written two-unit and two-file designs were reverted rather than left partial. |
| `e7530df` | Settled that interrupted fold. The commit pass 1 reviewed. |
| `91ec55e` | Pass 1's fix commit, by `claude-opus-5`. Re-applied both reverted designs in full and added the user's own `git/config` as a third consumer the plan had missed. |
| `6333542`, `90827b3`, `e5cc878` | Housekeeping between passes; `90827b3` moved the `nix.conf` append decision out of acceptance test 18 and into contract 4. `e5cc878` is the commit pass 2 reviewed. |
| `8bab682` | Pass 2's fix commit, by `claude-opus-5`. **Every claim in it is one model's fix for another model's finding, and nothing has verified it.** |

| Pass | Model | Effort | Reviewed commit | Findings | Repair commit | Survived later passes |
|---|---|---|---|---|---|---|
| 1 | `gpt-5.6-sol` | `xhigh` reported (see note) | `e7530df` (full pass) | 4 BLOCKER, 6 GAP, 1 SIMPLIFY. Origin: 8 original-plan, 2 introduced by `9e82896`, 1 introduced by `94c5ad6`. | `91ec55e` | Design held; text did not. Pass 2 verified all four blocker fixes and reversed none, but 6 of its 11 findings are defects in `91ec55e` itself. |
| 2 | `claude-fable-5` | unconfirmed (see note) | `e5cc878` (full pass over `e7530df..e5cc878`) | 0 BLOCKER, 10 GAP, 1 SIMPLIFY. Origin: 5 original-plan, 6 review-introduced — all but one from `91ec55e`, the last a consequence of its split. Findings 2 and 5 each split a facet across both origins. | `8bab682` | pending — no later pass yet |

**Effort is unconfirmed for both passes so far, for different reasons, and neither is a value to reuse.** Pass 1's runner was invoked with `model_reasoning_effort=high` while the reviewer self-reported `xhigh`; the reported value is recorded because a reported level is evidence of what applied and an invoked level is not. Pass 2's reviewer reported that it could not introspect the applied level at all, so nothing is recorded rather than the invoked value being promoted to an observation. `dev-plans.md` warns that an unsupported level can fall back silently rather than erroring, which is exactly why neither is inferred. Do not compare yields across these two passes as if effort were held constant, and confirm the level that actually applied before recording it next time.

Fix stability so far: `claude-opus-5` has authored the plan and both fix commits. Its structural fixes are stable — no design pass 1 or pass 2 accepted has been reversed — while its *prose and citation* work is not: `91ec55e` introduced six of pass 2's eleven findings, mostly wrong empirical claims and unwritable fixtures. That is the signal to review `8bab682` against, and it is why `claude-opus-5` is excluded from reviewing: it may not review its own work, and it is now the author of two rounds of fixes as well as the plan.

**Pass 3: recommend `gpt-5.6-terra`, full pass over `e5cc878..HEAD`** — `8bab682` plus this commit. Cross-vendor is the rotation `dev-plans.md` calls the strongest available, and it is affordable again now that pass 2 has spent the same-vendor slot. It is also the only rotation left that is fresh: `claude-opus-5` authored the plan and both folds, `gpt-5.6-sol` ran pass 1 and `claude-fable-5` ran pass 2. `dev-plans.md` bars `gpt-5.6-terra` when the prompt records repeated regressions in the feature under review; there are none — no accepted fix has been reversed by a later pass.

Full pass, not a scoped diff. Pass 2's yield was original-plan gaps plus fix-commit defects rather than "found little", so the coherence question — do Scope, Risk Assessment, the numbered contracts and the numbered tests still describe one design — is live across the whole plan, not only across the diff. Effort at least `xhigh`; `ultra` exists on this runner and is the exception the ladder describes, so justify it rather than defaulting to it, and confirm the level that actually applied before recording it. Reviews are launched manually, one model per pass; this is a recommendation for whoever starts pass 3, not a selection.

## Before Final Response

- Plan fixes are committed, or the pass explicitly found no plan changes.
- Focus Areas and the Review Evidence table are updated and committed.
- The final commit message includes the findings summary and `Model:` footer.
- The repo is pushed and `git status` is clean.
