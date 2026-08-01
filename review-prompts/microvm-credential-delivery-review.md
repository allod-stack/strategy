# microVM Runtime Credential Delivery Review Prompt

You have spent a career on the exact seam where a secret stops being ciphertext and starts being a file on a running machine. You know what `LoadCredential=` does when the credential is not there, you know which of `/run/credentials/@system` and `$CREDENTIALS_DIRECTORY` a unit can actually open, and you have personally been burned by a host key that regenerated itself at 3am and turned an anti-TOFU pin into a wall. You read NixOS module merge order the way other people read a shopping list, and you have a specific grudge against a security control whose only proof is that someone wrote it down.

This plan moves a VM's private key material off an encrypted-at-rest, decrypted-in-guest path onto a per-boot, host-memory path. It is the highest-risk change in its arc. Go break it. Do not be polite, and do not grade on effort.

## Your Task

Review [microvm-credential-delivery.md](../dev-plans/microvm-credential-delivery.md) for gaps, wrong premises, and landmines. Be direct and specific. Flag anything that blocks implementation, creates unnecessary work, or leaves a trap for the slices that follow.

**Read the actual code and the actual upstream sources.** This plan makes many empirical claims — what `sshd-keygen` does with an empty `hostKeys`, what agenix composes when `age.secrets` is `{}`, what systemd does with a missing `LoadCredential=`, where a fw_cfg credential lands in the guest, what QEMU's name-length limit is, whether a `lines` option needs `mkForce`. Verify them against the tree and against the pinned store paths. **A wrong empirical claim is the highest-value finding you can return.**

Pinned sources, all readable locally:

- microvm.nix `39a499ab85311b56dddb09ec43351cc3658f22c1` at `/nix/store/yyrsgw6gs5v1qnfn839mrn6mmj91zwfw-source` (narHash verified against `allod/vm` `flake.lock`).
- nixpkgs `b6018f87da91d19d0ab4cf979885689b469cdd41` at `/nix/store/4k79ns9drp02wyvcix81c7nnz4hn8psi-source` (narHash verified).
- agenix at `/nix/store/kg2mkqk4qkw5aqv1569cs5szhr71bc8y-source`.
- systemd 258.7 source at `/nix/store/a08jd53b59v5izippz0l5jpd61amra26-source`.
- QEMU 10.1.5 at `/nix/store/k8zfisphlb6sddc03r8p76xyga3s77wj-qemu-10.1.5` (source tarball `/nix/store/5cp2l2c344w5qf8b31fqnysscg1l2ylc-qemu-10.1.5.tar.xz`).

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

- `archetypes/flake.nix` is ~1879 lines. `sharedModules` at `:322` sets `services.openssh.hostKeys` and `age.identityPaths` to `/etc/ssh/<name>` for every guest — the SSH host key and the age identity are the same file.
- `microvmVolumesModule` at `:105-319` is the direct precedent: gated with `lib.optional (runtime == "microvm")` on the builder's module list, a typed option with a reused path validator, assertions over merged config. Its consuming check `runtime-module-selection` at `:1252-1738` has 16 sabotage fixtures, pins diagnostics through `config.assertions`, and probes the libvirt boundary with `lib.hasAttrByPath`.
- That check states its own memory constraint at `:1543-1554`: fixtures are consumed as thunks because holding them live is a measured OOM kill, and they take `nix flake check` from ~4 GB to over 5 on a 7 GiB box with **no swap**.
- Agents run in dev VMs. No host access, no real credentials, no boot.

Parent plan: [microvm-framework-adoption.md](../dev-plans/microvm-framework-adoption.md), converged at review pass 10. This slice implements contract **7**, the guest half of **8a**, and **9, 10, 11, 12, 14**. Do not re-litigate the parent's settled contracts — but if this plan or the code **contradicts** one, that is a finding. The plan already records two such contradictions; check whether they are the right calls and whether there are more.

## Structural Conformance

Verify every section `dev-plans.md` requires: Tracking Issue, Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates, Acceptance Tests, Rollback Plan. This is one PR, so per-PR risk does not apply. Verify the closing keyword: `Closes allod/archetypes#29`, and it must **not** close `allod/strategy#20`.

## Focus Areas

The six standing lenses in `dev-plans.md` apply as defaults. Concentrate here:

1. **Does the SSH host-key design actually prevent regeneration, and does it leave a bootable guest?** The plan claims `services.openssh.hostKeys = []` renders no `HostKey` line and gives `sshd-keygen` an empty script, so the key is pointed at via `services.openssh.extraConfig` instead. Verify all three halves against the pinned `nixos/modules/services/networking/ssh/sshd.nix`: the `HostKey` rendering, the `ConditionFileNotEmpty`/`script` behaviour with an empty list, and whether a plain `extraConfig` definition really lands after the module's own `lib.mkOrder 0` contribution. Then ask the harder question: is there **any** path — a profile, a Home Manager module, an upstream default, `startWhenNeeded`, the socket-activated `sshd@` unit — by which a host key gets minted or a second `HostKey` line appears? And does the build-time `sshd -G -T` config check tolerate a `HostKey` under `/run` that does not exist?

2. **Ordering, and what happens when the materializer fails.** The plan orders the unit before `sysinit.target` with `DefaultDependencies=no`, keeps an explicit `Before=sshd.service`, and makes sshd `Requires=` it. Is that correct and complete? What about the socket-activated `sshd@` path, `getty`, `systemd-logind`, Home Manager's user services, and anything that reads `/etc/nix/nix.conf` at socket-activation time? Conversely, is `Requires=` on sshd right, or does it create a failure mode worse than the one it closes — a materializer that fails on one optional credential taking down a guest that could otherwise have been reachable for repair? The plan explicitly accepts "sshd failed" as the correct outcome for the *host key*; is it also correct for the *Forge token*?

   **Open question carried forward, and this pass must answer it.** The plan author, mid-self-correction, began splitting this into **two** units — one for the SSH host key, one for every other credential — so that sshd's `Requires=` covers only the credential whose absence must stop sshd, and a failed Forge token no longer strands a guest that could have been reachable for repair. That change was announced in Scope and Risk Assessment but its contract text and acceptance tests were never written, so it has been reverted to the single-unit design rather than left half-applied. Decide it: one unit or two. If two, say exactly which credential goes in which unit, what each consumer `Requires=`, and what the paired sabotage is for the split itself — a two-unit design whose fixtures cannot tell the units apart is worse than one unit.

3. **The `lines`-option `mkForce` hazard, twice.** `nix.extraOptions` and `environment.etc."gitconfig".text` are both `types.lines` defined by `allod/vm` `modules/guest-base.nix`, and the plan overrides both with `mkForce`. `mkForce` on `lines` discards *every* other definition. Is the "exactly one line" generated-artifact check really sufficient to catch a future `allod/vm` change that adds a second, unrelated `extraOptions` setting which `mkForce` then swallows? Is there a less brittle shape — `nix.settings.netrc-file`, a separate `environment.etc` entry, an upstream change — and if so, is it worth the cross-repo cost? Say which you would do.

   **Open question carried forward, and this pass must answer it.** The author's interrupted self-correction was moving off `mkForce` in both places: for `nix.conf`, appending and asserting the *last* `netrc-file =` line wins while proving no unrelated `extraOptions` line disappeared; for Git, resetting the inherited `store` helper rather than replacing the whole file, and covering **both** `/etc/gitconfig` and the user's own `git/config` rather than only the system one. Neither reached the contracts or the acceptance tests, so both were reverted to the `mkForce` design. Decide each. The user `git/config` half is the substantive part and is worth checking on its own merits: the plan's consumer table names root's and the user's Git credential store as separate consumers, and test 19 currently reads one file.

4. **Is the credential-name set right, and is the derivation honest?** Four names, three of them conditional on exactly the values that already gate today's `age.secrets`. Check the conditions against `flake.nix:465`, `:480` and `machines.<name>.forge_key`. Is `forge-ssh-key` in the right slice at all, given `allod/nexus` `scripts/forge-ssh-key` installs it imperatively today and contract 18's rotation dispatch is a different slice — does declaring the name here strand something, or is leaving it out worse? And is the plan right to make a microvm machine with a non-empty `githubCredentialTargets` an evaluation error rather than deriving names for it?

5. **The consumer table's owners and modes.** `<root>` at `0711`, `netrc` and `git-credentials` at `0640 root:users`, the two private keys at `0600`. The plan argues the group-readable pair is not a new disclosure because today's `/home/<user>/.netrc` already gives the dev user the same bytes. Check that claim against the tree. Then check the modes against their consumers: will sshd accept the host key, will `ssh` accept the identity file, will the Nix daemon read the netrc, and does anything need the *public* half of a key that this table does not deliver?

6. **Can the checks actually fail?** Every assertion is supposed to have a sabotage fixture pinned to its diagnostic, verified by deletion. Walk the acceptance-test list and find the ones that would stay green with the thing they defend removed — particularly the generated-artifact greps, where an unanchored needle satisfied by unrelated text is the classic failure. The plan's own precedent records exactly this trap at `flake.nix:1402-1409`. Also: does the closure scan have a real positive control, or could it match nothing and pass?

7. **Memory.** The plan adds substantially more fixtures than the volumes slice, on a box where the existing check already peaks over 5 GB with no swap. Is the stated budget discipline enough, or does the fixture count need cutting before implementation starts? Name specific fixtures to merge or drop if so. Run a genuine SIMPLIFY sweep on the whole plan: what can be deleted outright?

8. **Risk calibration and the honesty of the gate.** The plan scores R3 and says the score does not come down with its own validation, because nothing here boots. Is R3 right, or is this R4 given it is a security-boundary change to how key material reaches a machine? Which premise would have to be wrong for your answer to flip? And is the "no machine is enabled on the strength of this change" gate stated in a way that actually binds, or is it a sentence someone will skip?

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

**No review pass has run.** The table is empty because it is accurate, not because a pass went unrecorded.

Three commits precede pass 1 and all three are the plan **author's own** work, so none of them counts as review and none of them has been checked by another model:

| Commit | What it is |
|---|---|
| `7ec057a` | The original plan and this prompt. |
| `9e82896` | Author self-correction: four claims that were inferences replaced with measured behaviour from the pinned nixpkgs, systemd 258.7 and OpenSSH sources — the empty-`hostKeys` fallback, the unloadable `sshd-keygen` unit, the soft-fail bare `LoadCredential=`, and `install(1)` not being atomic. Also corrected why contract 5 matters and moved ordering to `Before=sysinit.target`. |
| `94c5ad6` | Author self-correction, **interrupted by a session limit**. Its contract 1 and contract 3 edits are complete and were kept: the `pathErrors` split, the union-of-names rule, the comma rule on `credentialFiles` values, and the `types.path` note about a rule with no reachable fixture. Its Scope and Risk edits announced a two-unit materializer and a two-file Git design whose contract text and acceptance tests were never written; those were reverted rather than left half-applied, and both now sit in Focus Areas 2 and 3 as questions this pass must answer. |

Treat every claim in the plan as author-authored and unreviewed. In particular, the measured claims in `9e82896` are exactly the kind this prompt says are the highest-value findings — re-derive them from the pinned sources rather than trusting the commit message.

| Pass | Model | Effort | Reviewed commit | Findings | Fixing commit | Survived later passes |
|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — |

Plan authored by `claude-opus-5`. It may not review its own work; pass 1 must be a different model, and a cross-vendor rotation is the strongest available.

## Before Final Response

- Plan fixes are committed, or the pass explicitly found no plan changes.
- Focus Areas and the Review Evidence table are updated and committed.
- The final commit message includes the findings summary and `Model:` footer.
- The repo is pushed and `git status` is clean.
