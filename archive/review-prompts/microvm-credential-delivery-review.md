# microVM Runtime Credential Delivery Review Prompt

You have spent a career on the exact seam where a secret stops being ciphertext and starts being a file on a running machine. You know what `LoadCredential=` does when the credential is not there, you know which of `/run/credentials/@system` and `$CREDENTIALS_DIRECTORY` a unit can actually open, and you have personally been burned by a host key that regenerated itself at 3am and turned an anti-TOFU pin into a wall. You read NixOS module merge order the way other people read a shopping list, and you have a specific grudge against a security control whose only proof is that someone wrote it down.

This plan moves a VM's private key material off an encrypted-at-rest, decrypted-in-guest path onto a per-boot, host-memory path. It is the highest-risk change in its arc. Go break it. Do not be polite, and do not grade on effort.

## Your Task

Review [microvm-credential-delivery.md](../dev-plans/microvm-credential-delivery.md) for gaps, wrong premises, and landmines. Be direct and specific. Flag anything that blocks implementation, creates unnecessary work, or leaves a trap for the slices that follow.

**Read the actual code and the actual upstream sources.** This plan makes many empirical claims, and three passes have now shown that a written-as-measured claim is worth nothing until someone re-derives it. Pass 2 found one wrong claim in pass 1's fold. Pass 3 found another in pass 2's fold — nixpkgs' `commonUnitText` *does* emit `[Install] WantedBy=`, so "neither materializer's fragment gains a `WantedBy=` line" was false — **and it found two original-plan blockers that both earlier passes read past.**

That second half is the more important lesson, and it changes where to look. The first two passes concentrated on the newest text, because that is where the previous pass's defects were, and in doing so they walked past two design errors that had been in the plan since the first draft. The newest text is still the least reviewed, but "already reviewed twice" is not evidence of correctness.

The claims added by pass 3's fold (`77a6cea`) have been checked by nobody:

- Pinned git executes an **absolute-path** `credential.helper` verbatim and appends the operation as an argument, so a store-path helper needs no `!` prefix; and `git credential approve`/`reject` invoke a helper's `store`/`erase` unprompted after an authentication attempt.
- A `get`-only helper delegating to `git-credential-store` opens the credential file read-only and takes no lock, so it works from an unprivileged process inside a `0711 root:root` directory.
- Pinned systemd mounts `/run` with **no** `noswap` (`src/shared/mount-setup.c:98`) while `mount_credentials_fs()` (`src/shared/mount-util.c:1758`) requests it deliberately, which is the whole basis for contract 11.
- `nixpkgs`' `commonUnitText` emits `[Install] WantedBy=` for `wantedBy` and nothing at all for `requiredBy`, and the `.wants` symlink rather than the `[Install]` metadata is what starts an immutable NixOS unit.
- Test 19's helper-behaviour derivation and test 23's enumerate-filter-scan are both claimed to be affordable and both claimed to fail for their named reason. The memory-budget table claims a **net zero** forced-system change across the whole fold.

Verify against the tree and the pinned store paths. **A wrong empirical claim is the highest-value finding you can return, and an original-plan defect is worth as much — pass 3 proved they are still in there.**

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

Parent plan: [microvm-framework-adoption.md](../dev-plans/microvm-framework-adoption.md), converged at review pass 10. This slice implements contract **7**, the guest half of **8a**, and **11, 12, 14** in full, and **9** and **10** in part — pass 3 narrowed that claim, because the plan refuses the registered-GitHub-target path contract 9 requires and declines contract 10's consumer-failure clause for Forge credentials. Do not re-litigate the parent's settled contracts — but if this plan or the code **contradicts** one, that is a finding, and **record it in this plan's "Contract contradictions found" section rather than editing the parent**. The plan records five such contradictions. Pass 2 checked the first three and named each a right call; pass 3 checked the fourth and accepted it, and added the fifth. Check the fifth, check whether narrowing the tracking claim was the right response rather than implementing the GitHub path here, and check whether there are more — pass 3 found one that had gone unrecorded through two full passes.

## Structural Conformance

Verify every section `dev-plans.md` requires: Tracking Issue, Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates, Acceptance Tests, Rollback Plan. This is one PR, so per-PR risk does not apply. Verify the closing keyword: `Closes allod/archetypes#29`, and it must **not** close `allod/strategy#20`.

## Focus Areas

The six standing lenses in `dev-plans.md` apply as defaults. Concentrate here.

Three passes have run. Pass 1's four blockers changed the design; pass 2 verified that design end to end and found no blocker; **pass 3 then found two blockers that had been in the plan since its first draft**, plus four more original-plan gaps. Read that sequence before deciding where to spend attention: two independent frontier models read past a Git credential file wired to a helper that rewrites it, and past a missing assertion the parent plan states in one sentence. Anything in this plan can still be wrong, including text with three passes behind it.

1. **The read-only credential helper is a new design written by the fix author and checked by nobody.** Contract 4 no longer names `store --file=` in either Git config file. It generates a `get`-only wrapper, names it by absolute store path in `/etc/gitconfig` and in the user's `git/config`, and drains `store`/`erase`. Attack it from every side: does pinned git really execute an absolute-path helper verbatim, and does it really append the operation as a separate argument? Does the quoting difference between the raw `types.lines` file and Home Manager's `toGitINI` still come out to the same helper once git's config reader strips quotes? Does anything else on a microvm guest — a `forge` CLI path, a Home Manager activation step, a nix subprocess — still write that file? And is `erase` becoming a silent no-op acceptable, or does some consumer depend on a rejected credential being removed?

2. **Test 19's helper-behaviour derivation, which is the only place this plan measures rather than cites.** It stages a `0640` credential file, runs `git credential fill`, `approve` and `reject` against the generated helper, requires bytes and mode unchanged, and requires the *stock* `store --file=` helper to change the mode as a negative control. Verify that control actually fires on pinned git 2.51.2 — if `approve` does not rewrite the file in a sandbox, the whole blocker-1 rewrite rests on a documentation quote and the check is theatre. Verify the plan's own stated limit is honest: a build sandbox has one uid, so ownership is not measured here.

3. **Contract 11 and whether an evaluation assertion is the right control for swap.** The measurement is that `/run` is mounted without `noswap` while systemd's own credential filesystem asks for it. Check both readings against pinned systemd. Then ask the design question the contract records and declines: is asserting `swapDevices == []` and `!zramSwap.enable` the right control, or should `<root>` be its own `noswap` tmpfs? The contract rejects that on the grounds that it creates a second creator of the path and an ordering edge nothing here can boot to verify. Is that right, and are there other swap-shaped routes — a resume device, a swap file created at runtime, a profile-supplied `systemd.mounts` — the two asserted options miss?

4. **Sabotage-to-assertion assignment, swept over every fixture rather than at the one that was caught.** Pass 3 found test 13's relocation sabotage could not fail the disjointness and union assertions it named, because a true relocation leaves both true. **That is the fifth instance of this class in this arc, and all five were caught by a reviewer, never by a test.** The fold reassigned that sabotage and added a rule to the sabotage preamble: for every assertion a sabotage claims to break, the mutated configuration must actually falsify it. Check the rule was applied everywhere and not only where the finding pointed — walk each fixture in tests 6 to 15, 19 and 23, work out what the mutation makes false, and compare that against the needles it claims.

5. **Test 23's enumerate-filter-scan, which replaced a seven-artifact list.** It now enumerates every `system.activationScripts` entry, every unit fragment and `unit-script-*` derivation, and every Home Manager artifact; filters to those naming the credential root or a credential name; and scans the survivors. Two risks: the filter can drop an artifact that references a credential indirectly (through a variable, a wrapper, or a store path whose name does not contain the literal), and the enumeration can cost far more than the plan assumes because it forces artifacts nothing else forces. Verify both, and verify the sabotage — an unrelated activation script copying a credential under the dev home mount — is genuinely outside the old seven.

6. **The memory-budget net-zero claim, which is now a table rather than a sentence.** The plan claims the whole fold costs zero additional forced systems: three mutations ride the shared unit-mutation fixture, the swap sabotage rides test 10's, test 5's merge cancels the `wantedBy = []` thunk. It also claims test 14's two new sshd-branch fixtures are cheap because they render units without forcing `system.build.toplevel`. Check each row, and check the harder question the table dodges: can a single fixture carrying five unrelated mutations still pin each needle to its own assertion, or does one violation start masking another's diagnostic?

7. **The consumer table's modes against their real consumers. Three passes have not measured this half.** Pass 2 confirmed the `0640 root:users` pair is not a new disclosure and stopped; pass 3 then found the hazard one line away, in the helper that file was wired to. Treat "the mode was checked" as no evidence about anything else on that row. Still unmeasured: will sshd accept a host key at `0600 root:root` under `/run`, will `ssh` accept an `IdentityFile` at `0600 <user>:users`, will an unprivileged `nix` client actually open a `0640 root:users` netrc, and does anything need the *public* half of a key this table does not deliver?

8. **Coherence after a design change that touched four sections.** Blocker 1 moved through Scope, Risk Assessment, contract 4, contract 6's materializer, and tests 15, 19 and 23. Blocker 2 added contract 11 and touched Scope, Risk Assessment and test 10. Finding 3 changed what the Tracking Issue claims. Do Scope, Risk Assessment, the numbered contracts and the numbered tests describe **one** design now? This is the question pass 3's prompt asked and the fold's largest exposure.

9. **What three passes have read past.** The two blockers were both original-plan, so the sections nobody has attacked line by line are the place to look: contract 2's name/length table and its two independent gates, contract 3's value rules, contract 5's overlap property, contract 9's closure scan, and the libvirt no-op's four drvPath hashes. Do not assume a section is sound because no pass has filed against it.

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
- **Test 13 genuinely distinguishes a merged implementation from a split one**, and the no-attribute assertion on the privacy and no-Forge fixtures additionally forces the composed-outside-`mkIf` shape, since a `mkIf false` would still create the attribute. **Narrowed by pass 3, and the narrowing is the lesson**: distinguishing *merged from split* held, but the sabotage for a *mis-split* — `ssh-host-key` relocated onto the Forge unit — could not fail the disjointness and union assertions it named, because relocation leaves both true. A "settled" item covers exactly what the pass checked and not the adjacent thing it sounds like.
- **`After=systemd-sysusers.service` covers all three user-creation paths**: `userborn.service` carries the alias, `systemd.sysusers.enable` keeps the upstream unit name, classic activation creates users in stage 2 before any unit, and a nonexistent unit in `After=` is a no-op.
- **`Before=sysinit.target` covers every claimed consumer, sockets included** — verified in systemd 258.7's `service_add_default_dependencies` and `socket_add_default_dependencies`.
- **Reactivation needs both `requiredBy` and `before`**; pinned `userborn.nix` carries the target in both lists.
- **The no-`mkForce` append pair for `nix.conf`**, including last-occurrence-wins measured on Nix 2.31.5 and a preservation control that can genuinely fail.
- **The empty-string helper reset works across git's config file order**, not merely within one file (`gitcredentials.adoc:199-202`).
- **The double-rendering trap is real**, with one nuance kept in the plan: `multipleType`'s `either` merge means two *list* definitions concatenate rather than conflict, so test 19's ordered-list assertion — not a merge error — is what pins the final helper sequence.
- **`hostKeys = []` as a lexical definition rather than an omission**, `sshd-keygen.enable = false` rendering a real `/dev/null` mask, and `HostKey` as a list under `settings` throwing.
- **The `0640 root:users` pair is not a new disclosure**, and `forgeKey ? machines.${name}.forge_key` read from `machines` rather than `vmFacts` is the right shape. **Read the first half narrowly**: the *mode* was checked and is fine; the helper that file was wired to was not, and pass 3 found the blocker there. A checked mode is no evidence about the rest of its row.
- **All recorded contract contradictions are the right calls**, including contradiction 3 re-verified end to end.

### Settled by pass 3 — same rule

`gpt-5.6-sol` re-derived these against the pinned sources and named each correct. Reversing one needs a measurement.

- **The four systemd symlink names and their location** under `config.environment.etc."systemd/system".source`, spelled as `generateUnits` emits them.
- **All three sshd edges belong under `services.openssh.enable`**, with `sshd.service` and `sshd.socket` additionally gated on `startWhenNeeded` in opposite directions and `sshd@` gated on neither.
- **The defaulted `githubCredentialTargets` threading is drvPath-neutral** for every existing `mkDevVm` call, and an injected target reaches the microvm refusal.
- **Test 2's `devFixture // args` route works**, carrying an overridden `identity` and a `forgeKey` through to `mkDevVm`.
- **Repeated `[credential]` sections are legal git config**, so the appended block's own header costs nothing.
- **Nix 2.31.5 runs the `git` binary as a subprocess for `git+https`**, which is the mechanism the whole one-netrc argument rests on. Pass 3 also confirmed the pinned sources do **not** establish that `~/.netrc` has stopped working, and that the plan correctly avoids claiming it.
- **Contradiction 4 is the right call** — the two-unit split's deviation from parent contract 10 is correctly recorded rather than worked around.

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

**Three review passes have run, and the plan is not converging on the schedule pass 2 suggested.** Pass 1 found four blockers and changed the design. Pass 2 found none, which read at the time as evidence the design was right. Pass 3 then found **two blockers, both original-plan**, which retires that reading: a clean pass from one model is evidence about that model's coverage, not about the plan.

**The stopping rule state is plain: the consecutive-clean-pass count is zero.** Pass 3 produced two blockers and six original-plan gaps, so it is not a clean pass and does not start the count. The other stopping condition also sits at zero: pass 3's findings split six original-plan to six review-introduced, and six does not outnumber six, so the "review-introduced outnumber original-plan for two consecutive passes" counter reset rather than advancing to two.

**The rotation result is the strongest evidence in this plan's history for rotating, and it should be read that way.** Pass 2 (`claude-fable-5`, a frontier model at a high effort) returned no blockers over a full pass. Pass 3 (`gpt-5.6-sol`, a different vendor) then found two original-plan blockers that pass 1 *and* pass 2 had both read past — a Git credential file wired to a helper that rewrites it, and a missing assertion for a boundary the parent plan states in one sentence. Neither was subtle once named. Whatever a single model's clean pass means, it does not mean the plan is done.

Commit provenance, which is what the origin column below refers to:

| Commit | What it is |
|---|---|
| `7ec057a` | The original plan and this prompt. |
| `9e82896` | Author self-correction: four inferences replaced with measured behaviour from pinned nixpkgs, systemd and OpenSSH, and ordering moved to `Before=sysinit.target`. |
| `94c5ad6` | Author self-correction interrupted by a session limit. Its contract 1 and 3 edits were kept; its half-written two-unit and two-file designs were reverted rather than left partial. |
| `e7530df` | Settled that interrupted fold. The commit pass 1 reviewed. |
| `91ec55e` | Pass 1's fix commit, by `claude-opus-5`. Re-applied both reverted designs in full and added the user's own `git/config` as a third consumer the plan had missed. |
| `6333542`, `90827b3`, `e5cc878` | Housekeeping between passes; `90827b3` moved the `nix.conf` append decision out of acceptance test 18 and into contract 4. `e5cc878` is the commit pass 2 reviewed. |
| `8bab682` | Pass 2's fix commit, by `claude-opus-5`. Pass 3 checked it and reversed one claim (`commonUnitText` does emit `[Install] WantedBy=`). |
| `756e64f` | Recorded pass 2 and pointed the focus areas at what it left. The commit pass 3 reviewed. |
| `77a6cea` | Pass 3's fix commit, by `claude-opus-5`. Carries both blocker rewrites — the read-only Git credential helper and contract 11's no-swap assertion — plus all ten other findings. **Every claim in it is one model's fix for another model's finding, and nothing has verified it.** |

| Pass | Model | Effort | Reviewed commit | Findings | Repair commit | Survived later passes |
|---|---|---|---|---|---|---|
| 1 | `gpt-5.6-sol` | `xhigh` reported (see note) | `e7530df` (full pass) | 4 BLOCKER, 6 GAP, 1 SIMPLIFY. Origin: 8 original-plan, 2 introduced by `9e82896`, 1 introduced by `94c5ad6`. | `91ec55e` | Design held; text did not. Pass 2 verified all four blocker fixes and reversed none, but 6 of its 11 findings are defects in `91ec55e` itself. |
| 2 | `claude-fable-5` | unconfirmed (see note) | `e5cc878` (full pass over `e7530df..e5cc878`) | 0 BLOCKER, 10 GAP, 1 SIMPLIFY. Origin: 5 original-plan, 6 review-introduced — all but one from `91ec55e`, the last a consequence of its split. Findings 2 and 5 each split a facet across both origins. | `8bab682` | Design held. Pass 3 narrowed one accepted item (test 13 distinguishes merged from split, but its mis-split sabotage could not fail) and reversed one claim in its fix commit. |
| 3 | `gpt-5.6-sol` | invoked `xhigh`, unconfirmed (see note) | `756e64f` (full pass) | 2 BLOCKER, 9 GAP, 1 SIMPLIFY. Origin: 6 original-plan (findings 1-6, including both blockers), 6 review-introduced — 3 from `8bab682`, 3 from `91ec55e`. | `77a6cea` | pending — no later pass yet |

**Effort is unconfirmed for all three passes, and the pattern is now the finding.** Pass 1's runner was invoked with `model_reasoning_effort=high` while the reviewer self-reported `xhigh`; the reported value is recorded because a reported level is evidence of what applied and an invoked level is not. Pass 2's reviewer could not introspect the applied level at all. Pass 3 was invoked with `model_reasoning_effort=xhigh` and its reviewer reported it could introspect **neither the deployed model id nor the effort** — only that it was in the Codex/GPT-5 family — so the row above records invoked values, marked unconfirmed, rather than promoting them to observations.

**Record that as a limitation rather than papering over it.** `dev-plans.md` asks for the exact effort in every pass, asks that effort be held constant when comparing model stability, and warns that an unsupported level can fall back silently instead of erroring. Two of three passes could not confirm their own effort and one could not confirm its own model id, so the roster's fix-stability tracking for this plan rests on invocation records that nothing inside the session can corroborate. Treat cross-pass yield comparisons here as unreliable, and if a future runner exposes the applied level, say so explicitly in the pass output — that is the cheapest thing that would make this table mean what it claims.

Fix stability so far: `claude-opus-5` has authored the plan and all three fix commits. Its *structural* fixes remain stable — no design accepted by a later pass has been reversed, and pass 3 accepted the two-unit split, the reactivation wiring and the two-file Git rewiring. Its *prose, citation and test-design* work is not stable: `91ec55e` introduced six of pass 2's eleven findings and three more of pass 3's, and `8bab682` introduced three of pass 3's, including one flatly wrong empirical claim written as measured. Two folds in a row have introduced defects into the same feature's text. That is the record to weigh below, and it is why `claude-opus-5` is excluded from reviewing: it may not review its own work, and it now authors three rounds of fixes as well as the plan.

**Pass 4: recommend `gpt-5.5` at `xhigh`, full pass over `756e64f..HEAD`** — `77a6cea` plus this commit.

The recommendation comes with a roster tension worth stating, because the R3-appropriate tier is nearly exhausted. `claude-opus-5` is the author and excluded. `gpt-5.6-sol` has run twice. `claude-fable-5` has run once and returned the pass that missed both blockers. That leaves three eligible rows, and none is an obvious default:

- `gpt-5.6-terra` is the next row down and is cross-vendor and fresh, but its roster role is "R0-R2 plans and an independent rotation partner for broader first passes" — a tier below what an R3 plan that just produced two blockers wants — and `dev-plans.md` bars it when the prompt records repeated regressions in the feature under review. Read strictly, two consecutive folds introducing defects into the same feature's text is that record, even though no *design* has regressed. Both halves point away from it here.
- `claude-opus-4-8` is eligible and its roster role is exactly "scoped stability pass on a fix authored by Opus 5", which this fold is. But the pass needed is a full one, not a scoped diff, and it is the same vendor and family as the fix author — the weakest rotation on offer, when the evidence from pass 3 is that a *cross-vendor* swap is what found what two same-generation frontier models missed.
- `gpt-5.5` is the previous frontier model with an `xhigh` ceiling, cross-vendor from the fix author, and its roster row states its own precondition: "rotation partner once the 5.6 family has already reviewed the feature." That precondition is now met twice over. It has never seen this plan, so it carries none of the anchoring the three prior reviewers now have.

**So `gpt-5.5`, for the reason rather than by elimination**: it is the strongest cross-vendor option whose roster condition is satisfied, and cross-vendor rotation is the only thing in this plan's history that has surfaced an original-plan blocker after a clean pass. Effort `xhigh`, its ceiling. Confirm the applied level and the model id before recording them, and if the runner cannot report either, say so in the pass output rather than leaving the row to be filled in from the invocation.

Full pass, not a scoped diff. Pass 3's yield was half original-plan, including both blockers, so a diff-scoped review would look at exactly the text least likely to hold the next blocker. The coherence question is also live across the whole plan and not only the diff: blocker 1 moved through Scope, Risk Assessment, contract 4, contract 6 and three tests.

**And note where this ends.** After pass 4, the R3-appropriate tier is spent. If the plan has not produced two consecutive clean passes by then, the honest options are to stop reviewing text and start implementing — the acceptance tests are where the remaining doubt gets settled — or to re-run a model at a different effort while recording plainly that `dev-plans.md` does not count that as independent rotation. Do not fill the slot by promoting a checklist-tier model into a terminal review of an R3 security-boundary plan.

## Before Final Response

- Plan fixes are committed, or the pass explicitly found no plan changes.
- Focus Areas and the Review Evidence table are updated and committed.
- The final commit message includes the findings summary and `Model:` footer.
- The repo is pushed and `git status` is clean.
