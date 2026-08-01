# microVM Runtime Credential Delivery Review Prompt

You have spent a career on the exact seam where a secret stops being ciphertext and starts being a file on a running machine. You know what `LoadCredential=` does when the credential is not there, you know which of `/run/credentials/@system` and `$CREDENTIALS_DIRECTORY` a unit can actually open, and you have personally been burned by a host key that regenerated itself at 3am and turned an anti-TOFU pin into a wall. You read NixOS module merge order the way other people read a shopping list, and you have a specific grudge against a security control whose only proof is that someone wrote it down.

This plan moves a VM's private key material off an encrypted-at-rest, decrypted-in-guest path onto a per-boot, host-memory path. It is the highest-risk change in its arc. Go break it. Do not be polite, and do not grade on effort.

## Your Task

Review [microvm-credential-delivery.md](../dev-plans/microvm-credential-delivery.md) for gaps, wrong premises, and landmines. Be direct and specific. Flag anything that blocks implementation, creates unnecessary work, or leaves a trap for the slices that follow.

**Read the actual code and the actual upstream sources.** This plan makes many empirical claims. The ones added since pass 1 have been checked by nobody: that `userborn.service` carries `aliases = [ "systemd-sysusers.service" ]` so one `After=` covers both user-creation paths; that `pkgs.formats.nixConf` interpolates `extraOptions` after every `settings` key, which is what rules out `nix.settings.netrc-file`; that `lib.generators.toGitINI` quotes every string value while `environment.etc.<n>.text` does not; that Home Manager's `gitIniType` renders a list as one line per element; that git's empty-string helper resets the list; and that nixpkgs defines `systemd.services.sshd` under `mkIf (!startWhenNeeded)`. The older ones — what `sshd-keygen` does with an empty `hostKeys`, what agenix composes when `age.secrets` is `{}`, what systemd does with a missing `LoadCredential=`, where a fw_cfg credential lands, QEMU's name-length limit — were re-derived by pass 1 and held. Verify against the tree and the pinned store paths. **A wrong empirical claim is the highest-value finding you can return.**

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

Parent plan: [microvm-framework-adoption.md](../dev-plans/microvm-framework-adoption.md), converged at review pass 10. This slice implements contract **7**, the guest half of **8a**, and **9, 10, 11, 12, 14**. Do not re-litigate the parent's settled contracts — but if this plan or the code **contradicts** one, that is a finding, and **record it in this plan's "Contract contradictions found" section rather than editing the parent**. The plan records three such contradictions; the third, added in `91ec55e`, is the empirical result that no normalization step exists anywhere on this path for parent contract 7's "duplicate normalized name" to collide through. Check whether all three are the right calls and whether there are more.

## Structural Conformance

Verify every section `dev-plans.md` requires: Tracking Issue, Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates, Acceptance Tests, Rollback Plan. This is one PR, so per-PR risk does not apply. Verify the closing keyword: `Closes allod/archetypes#29`, and it must **not** close `allod/strategy#20`.

## Focus Areas

The six standing lenses in `dev-plans.md` apply as defaults. Concentrate here.

Pass 1 closed three of the questions this section used to carry — one unit or two, `mkForce` on `nix.extraOptions`, and whether the user's own `git/config` needs covering. They are answered below as settled design, not reopened. What replaces them is verification: pass 1's four blockers all changed the design, and a design changed by a review has not been reviewed.

1. **The two-unit split, which is new in `91ec55e` and is the largest structural change in the plan.** Contract 6 now specifies `allod-microvm-host-key.service` (always composed, owns `ssh-host-key` and creates the credential root, is what sshd `Requires=`) and `allod-microvm-forge-credentials.service` (composed only when the machine declares a Forge name, owns everything else, required by nothing). Break it:

   - Is the dependency direction between them right? The Forge unit `Requires=`/`After=` the host-key unit so the root has one creator. Does that recreate any stranding, or leave the root uncreated on a path where the host-key unit is skipped rather than failed?
   - `After=systemd-sysusers.service` on the Forge unit, justified by pinned `userborn.service` carrying `aliases = [ "systemd-sysusers.service" ]`. Verify the alias, and verify the classic non-sysusers path really does create the user before any `DefaultDependencies=no` unit runs. If it does not, the plan has a race it claims it does not.
   - Does `Before=sysinit.target` on both actually cover every consumer the plan claims — `getty`, `systemd-logind`, `user@.service`, Home Manager's user services, `nix-daemon.socket`? The plan asserts it and cites the default `After=sysinit.target`; test it against a real rendered unit graph rather than the manual page.
   - Are the fixtures able to tell the two units apart, or would a merged-unit implementation pass them? Test 13's disjointness and union assertions are the whole defence.

2. **Does the SSH host-key design leave a bootable guest?** Pass 1 confirmed the anti-regeneration half: `hostKeys = []` plus one explicit runtime `HostKey` plus a masked `sshd-keygen` prevents minting, and that is settled — do not re-derive it. What is still open is everything around a *second* key appearing: is there any path — a profile, a Home Manager module, an upstream default, `startWhenNeeded`, the socket-activated `sshd@` unit — by which a host key gets minted or a second `HostKey` line appears? Does the build-time `sshd -G -T` config check tolerate a `HostKey` under `/run` that does not exist? And check the new conditional sshd wiring: `91ec55e` gates it on `config.services.openssh.startWhenNeeded` because nixpkgs defines `systemd.services.sshd` under `mkIf (!startWhenNeeded)`, so an unconditional `requires` would materialize a unit nixpkgs deliberately renders none of. Verify that gating is written the way sshd's own module writes it.

3. **The two Git config files, and the double-rendering trap.** Settled: both files get an empty-string helper reset followed by the runtime helper, no `mkForce` anywhere, and `modules/home-shared.nix` takes a `credentialRoot` argument so the user's file is covered too. What to verify: does the reset actually clear a lower-priority helper in *git's* read order, or only within one file? `/etc/gitconfig` renders unquoted through `types.lines` while Home Manager renders quoted through `lib.generators.toGitINI`'s `mkValueString` — does test 19 read each file in its own form, and would it catch a fixture that fixed only the system file? Is `credential.helper = [ "" "…" ]` really rendered as two ordered lines by the pinned Home Manager `gitIniType`, and does a second definition of that attribute conflict rather than merge?

4. **`nix.conf`: does the preservation control actually preserve?** Settled: append with `lib.mkAfter`, assert the *last* `netrc-file =` occurrence is the runtime path, and inject one unrelated `extraOptions` line as a control. `nix.settings.netrc-file` is ruled out because `pkgs.formats.nixConf` interpolates `extraOptions` after every `settings` key, so the inherited legacy line would win — verify that in the pinned generator. Then the real question: can the control fail? Show what change to `allod/vm` the injected line would catch, and confirm Nix's own parser takes the last occurrence of a repeated setting rather than the first.

5. **The credential-name set, now that both gates are separate.** Pass 1 found the plan asserting a machine shape that cannot exist. `91ec55e` splits the gates — the two token files gate the API/HTTPS names, `machines.<name>.forge_key` independently gates `forge-ssh-key` — and adds `forgeKey ? machines.${name}.forge_key` to `mkDevVm` so a fixture can reach the no-Forge-at-all shape. Verify the three fixtures in test 2 against the actual builder and inventory, and verify the new argument changes no libvirt derivation. `forge-ssh-key` staying in this slice is settled (parent contract 9 moves its `IdentityFile` off persistent home); the `githubCredentialTargets` evaluation error is unchallenged and can stay closed.

6. **The consumer table's owners and modes. Pass 1 returned no finding here, which is not the same as clearing it — treat it as unexamined.** `<root>` at `0711`, `netrc` and `git-credentials` at `0640 root:users`, the two private keys at `0600`. The plan argues the group-readable pair is not a new disclosure because today's `/home/<user>/.netrc` already gives the dev user the same bytes. Check that claim against the tree. Then check the modes against their consumers: will sshd accept the host key, will `ssh` accept the identity file, will the Nix daemon read the netrc, and does anything need the *public* half of a key that this table does not deliver?

7. **Can the checks actually fail, now that most of them are not fixtures?** Pass 1 deleted two sabotages the plan proved could not exist (a relative `credentialFiles` value, two names colliding after normalization) and found the closure scan's positive control validated only one of its three detectors. `91ec55e` restructures the whole list: exhaustive pure-validator tables carry the shape coverage, and each assertion *family* keeps one full-NixOS fixture whose only job is to prove the validator is wired to the merged value. Attack that: is the table genuinely exhaustive, or did a rule lose its only case in the move? Does each surviving wiring fixture actually pin the assertion it names — the trap the existing `imageRootOption` comment records, where an unanchored needle is satisfied by a different reporter's message? And do the three closure detectors each now reject their own canary?

8. **Memory, again, because the fixture list was rebuilt.** The existing check peaks over 5 GB on a 7 GiB box with no swap. `91ec55e` cuts the full-system fixture count and requires measuring after each small batch rather than after the list. Is the family list complete — is there an assertion with no wiring fixture at all now? Is the ~5% per-batch stop rule workable, or is it ceremony? Run a genuine SIMPLIFY sweep on the whole plan: what can be deleted outright?

9. **Risk calibration, and the citation policy.** R3 is settled: pass 1 confirmed it is correctly calibrated because no machine selects this path and the consumption gate requires later nested-boot evidence, and recorded the escalation trigger — it becomes R4 if this PR enables a deployed machine or is accepted as sufficient evidence for cutover. Do not re-litigate the score; do check that the gate is still stated in a way that binds after the Risk section was edited. Separately, pass 1 found three plan citations pointing at unrelated code because `archetypes/flake.nix` grew from ~1879 to over 2000 lines. The plan now cites working-tree constructs by identifier and quoted text, and keeps line numbers only for pinned store paths. Spot-check that every citation still resolves, and say so if the policy itself is being followed inconsistently.

### Settled by pass 1 — do not undo without new evidence

`gpt-5.6-sol` examined these and named each as correct. Re-deriving them is wasted effort; *reversing* one needs a measurement, not an opinion.

- The absolute-path `LoadCredential=<name>:/run/credentials/@system/<name>` form is correctly fail-closed, and the bare form's soft-fail is the reason.
- `hostKeys = []` plus one explicit runtime `HostKey` plus a masked `sshd-keygen` correctly prevents regeneration.
- Keeping the agenix module imported with `age.secrets == {}` is sound — it keeps the option declared so checks can assert `== {}` rather than probe for absence.
- Same-directory temporary writes followed by `mv -T` are sound.
- Rejecting `systemd-tmpfiles` is justified, for the three reasons contract 6 records.
- R3 is correctly calibrated. It becomes R4 if this PR enables a deployed machine or is accepted as sufficient evidence for cutover.

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

**One review pass has run.** It found four blockers, so the plan a second pass reads is materially different from the one pass 1 read.

Four commits precede pass 2. The first three are the plan **author's own** work and count as authoring, not review:

| Commit | What it is |
|---|---|
| `7ec057a` | The original plan and this prompt. |
| `9e82896` | Author self-correction: four claims that were inferences replaced with measured behaviour from the pinned nixpkgs, systemd 258.7 and OpenSSH sources — the empty-`hostKeys` fallback, the unloadable `sshd-keygen` unit, the soft-fail bare `LoadCredential=`, and `install(1)` not being atomic. Also corrected why contract 5 matters and moved ordering to `Before=sysinit.target`. |
| `94c5ad6` | Author self-correction, **interrupted by a session limit**. Its contract 1 and contract 3 edits were kept: the `pathErrors` split, the union-of-names rule, the comma rule on `credentialFiles` values, and the `types.path` note about a rule with no reachable fixture. Its Scope and Risk edits announced a two-unit materializer and a two-file Git design whose contract text and acceptance tests were never written; those were reverted rather than left half-applied. |
| `e7530df` | Settled that interrupted fold and carried the two reverted designs into Focus Areas as open questions. This is the commit pass 1 reviewed. |

`91ec55e` is pass 1's fix commit, authored by `claude-opus-5` from `gpt-5.6-sol`'s findings. Both of `94c5ad6`'s reverted designs were re-applied there, this time complete: pass 1 independently arrived at the two-unit materializer and at dropping `mkForce` for both `nix.extraOptions` and the Git helper, and added the user's own `git/config` as a third consumer the plan had missed. **Every claim in `91ec55e` is one model's fix for another model's finding and nothing has verified it.**

The measured claims in `9e82896` were re-derived by pass 1 and held. The measured claims added since — the `userborn.service` alias, the `nixConf` generator's ordering of `extraOptions`, the `toGitINI` quoting, the `mkIf (!startWhenNeeded)` guard on `systemd.services.sshd` — have not been checked by anyone.

| Pass | Model | Effort | Reviewed commit | Findings | Fixing commit | Survived later passes |
|---|---|---|---|---|---|---|
| 1 | `gpt-5.6-sol` | `xhigh` reported (see note) | `e7530df` (full pass) | 4 BLOCKER, 6 GAP, 1 SIMPLIFY. Origin: 8 original-plan, 2 introduced by `9e82896`, 1 introduced by `94c5ad6`. | `91ec55e` | pending — no later pass yet |

**Effort discrepancy, recorded because the recorded value must be the one observed.** The runner was invoked with `model_reasoning_effort=high`; the reviewer self-reported `xhigh`. `dev-plans.md` warns that a level the model does not support can fall back silently rather than erroring, so a reported level is evidence of what applied and an invoked level is not. `xhigh` is recorded as the reported value with the discrepancy attached. Do not compare pass 1's yield against a later pass as if the effort were confirmed, and confirm the level that actually applied before recording it next time.

Plan authored by `claude-opus-5`, which also authored `91ec55e`. It may not review its own work, and it is now the author of the fixes as well as the plan, so it is excluded from pass 2 on both counts.

**Pass 2: recommend `claude-fable-5` at `xhigh`, full pass over `e7530df..HEAD`** — `91ec55e` (the fixes), `6333542` (this prompt) and `90827b3` (contract 4 gains the `nix.conf` append decision that had been left in acceptance test 18 alone). Two models are excluded and neither exclusion is about quality: `claude-opus-5` authored the plan and the fixes, and `gpt-5.6-sol` ran pass 1, so a second `gpt-5.6-sol` pass would be grading its own findings rather than rotating. `claude-fable-5` is the roster's other R4-class option and has not seen this plan. `xhigh` is the right level for a security-boundary change with cross-repo generated lifecycle behaviour, and pass 1's effort is itself uncertain (see the note above), so holding effort constant is not available as a comparison anyway.

This costs the cross-vendor rotation for one pass, which `dev-plans.md` calls the strongest available. Take it back at pass 3 — most naturally with `gpt-5.6-terra`, which has not reviewed this feature — unless pass 2 finds little, in which case go cross-vendor immediately rather than running a third same-vendor pass. Reviews are launched manually one model per pass; this is a recommendation for whoever starts pass 2, not a selection.

## Before Final Response

- Plan fixes are committed, or the pass explicitly found no plan changes.
- Focus Areas and the Review Evidence table are updated and committed.
- The final commit message includes the findings summary and `Model:` footer.
- The repo is pushed and `git status` is clean.
