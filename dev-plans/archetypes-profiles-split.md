# Archetypes/Profiles Repo Split

## Tracking Issue

To be created on `allod/strategy` before the first implementation PR opens —
filing it is the first agent action of this plan. Open the issue body with this
user story, then update this section with the issue URL:

> So that every piece of configuration lives in the one repo meant to own it —
> and nothing operator-private can end up in the public framework — I want the
> framework repo, the example machine configs, and the secrets template split so
> that reusable behavior, machine definitions, and identity data each have
> exactly one owning repo.

Multi-PR work. Every PR carries `Refs allod/strategy#<N>`; the final sweep PR
(M5, `allod/memory`) carries `Closes allod/strategy#<N>`.

## Goal

Rename the framework repo `allod/profiles` to `allod/archetypes`, move machine
profile definitions into a single dedicated `profiles` flake input (public
example repo `allod/profiles`, redirectable by a deploy flake to an operator's
own definitions repo), return the `secrets` template to identity-only exports,
and adapt the public `allod/deploy` template to the split — with a generic
composed-layer canary so a deploy flake fails loudly when its profiles
redirect is lost.

## Context

Current state, all visible in the public tree:

- The framework repo (today `allod/profiles`) reads machine profile definitions
  from three homes: its own `hosts/` tree (the in-flake
  `publicProfileDefinitions` attrset), the `secrets` input
  (`secrets.lib.profileDefinitions`), and the `inventory` input
  (`inventory.lib.profileDefinitions`, which no inventory exports — a dead
  seam). Three homes for one fact type invites drift; one source of truth per
  fact, consumers derive.
- The `secrets` template exports behavior in violation of its own charter:
  `lib.profileDefinitions = {}`, `lib.profileData = {}`, and
  `homeModules.preferences` (a real Home Manager module in
  `modules/preferences.nix`). Secrets own identity and trust roots, not machine
  behavior.
- The framework's core abstraction is the archetype
  (`profileArchetypes = [ "dev" "privacy" "hypervisor" ]`; one concrete machine
  per archetype). The name `profiles` describes machine configuration — which
  the repo should not own. The rename gives each name to the repo it describes.
- `allod/deploy` holds the pre-split thin adapter (allod/deploy#1): it pins
  `profiles`, `secrets`, and `inventory`, follows-redirects the two data
  inputs at the synthetic templates, and re-exports `nixosConfigurations` and
  `vmFacts`. Its lock pins the framework at the `allod/profiles` URL — which
  makes the template itself the one public consumer the rename cliff can
  break, handled in-plan at G1.
- Public provisioning entry points are redirect-cliff consumers too:
  inventory-driven cold clones, `DEPLOY_FLAKE` defaults, and rotation runbook
  lock-bump steps must stop using the old `allod/profiles` framework checkout
  before M1 claims that name for the definitions repo.

Target shape:

| Name | Public repo (`allod/`) | Operator fork / redirect |
|------|------------------------|--------------------------|
| `archetypes` | framework: archetype merge, builders, shared modules, `vmFacts`, checks — renamed from `allod/profiles` | consumed as an input; no fork needed |
| `profiles` | synthetic example machine definitions, one per archetype — new repo, fresh history | operator's real machine definitions, same export contract |
| `deploy` | thin composition root template — adapted to the split | operator's composition root: pins + input redirects |
| `secrets` | synthetic identity template — behavior exports removed | operator's real identity data |
| `inventory` | synthetic machine facts — pointer/registry updates only | operator's real facts |

`vm`, `nexus`, `tools`, `memory`, `strategy`: no renames. `nexus`,
`inventory`, and `memory` receive mechanical pointer and doc updates in the
sweep milestone.

## Scope

In scope:

- Forge-level rename `allod/profiles` -> `allod/archetypes` (human-only, M0).
- New repo `allod/profiles` with fresh history (human creates the empty repo,
  M1; agent populates): the example machine definitions currently in the
  framework's `hosts/` tree, the preferences Home Manager module currently in
  the secrets template's `modules/preferences.nix`, a flake exporting the
  profiles-input contract (Interface Contracts 1), standing repo hygiene
  (`LICENSE` copied from `allod/deploy` (GPL-3.0-or-later; org licensing is
  tracked by allod/strategy#11), `README.md` carrying the per-profile ownership
  statement that today lives in the framework README, `hooks/commit-msg`,
  `setup.sh`, `.gitignore`).
- `allod/archetypes` seam flip (M2, one PR): add the `profiles` input; source
  profile definitions, per-machine `profileData`, and the preferences module
  only from it; delete the in-tree `publicProfileDefinitions`, the `hosts/`
  example definitions, and the `secrets`/`inventory` definition reads; relocate
  `hosts/dev/home-shared.nix` (shared dev-archetype behavior, imported by
  `mkDevVm` for every dev VM — framework, not example) to `modules/`; export
  the compose helper, `profilesSource`, and the composed-layer check builder
  (Interface Contracts 2); rewrite `README.md` (name, ownership, history
  note).
- G1 public redirect-cliff re-points before M1: the `allod/deploy` URL-only
  re-point, plus the `inventory`/`nexus` runtime pointers that are already
  valid pre-split — `DEPLOY_FLAKE` defaults and lock-bump runbook steps point
  at `allod/deploy`, and inventory's framework checkout points at
  `allod/archetypes` instead of relying on the soon-dead `allod/profiles`
  redirect. Optional post-provision hook lookup remains a profiles-definitions
  concern, not a deploy-flake concern.
- `allod/deploy` adaptation (M3): rename the framework input `profiles` ->
  `archetypes`, add the `profiles` definitions input with its
  follows-redirect, add the composed-layer canary as a flake check plus a
  sabotage fixture, and update the README. (The G1-time URL-only re-point is
  a separate, earlier PR — see Sequencing.)
- `secrets` template cleanup (M4): remove `lib.profileDefinitions`,
  `lib.profileData`, `homeModules.preferences`, and `modules/preferences.nix`;
  README ownership touch-up.
- Sweep (M5): add the new definitions repo checkout where useful, finish
  `inventory` registry alias cleanup and `vm-specs.json` regen, update `memory`
  topic files, and do the grep-driven cleanup of remaining `allod/profiles`
  references across public repos. Redirect-sensitive provisioning defaults and
  framework checkouts are not deferred here; G1 owns them.

Out of scope:

- Operator-side migration: real machine definition content, per-machine NixOS
  `assertions` for machine invariants, deploy-fork adoption, and full-fleet
  composition parity. A companion plan in the operator's private workspace owns
  that integration and links here; this plan is a self-contained public leaf.
- Machine-specific policy anywhere in `archetypes` or `deploy`. The canary is
  generic (layer provenance only). Per-machine invariants belong inside that
  machine's profile modules as ordinary NixOS `assertions` — that is the
  supported pattern, implemented by whoever owns the machine.
- Restructuring `nexus` rotation runbooks beyond mechanical pointer updates.
  Re-basing rotation docs on deploy-flake concepts more deeply is follow-up
  work if ever wanted.
- The service archetype. `mkServiceVm` stays TBD; the `hosts/service/.gitkeep`
  placeholder is dropped, not moved — recreate the directory when the first
  service VM exists.
- Org-wide licensing (allod/strategy#11). New repos copy the existing
  GPL-3.0-or-later LICENSE; changing licenses is that issue's business.
- Anonymous-HTTPS availability of org repos (forge visibility settings).

## Sequencing and Landing Order

The rename mechanics are the load-bearing constraint. A Forgejo rename leaves a
redirect: the old URL keeps serving the renamed repo — until a new repo claims
the old name, which kills the redirect permanently. Sequence around that cliff:

1. **M0 (human)**: rename `allod/profiles` -> `allod/archetypes` on the forge.
   Do not create the replacement repo in the same session. Everything keeps
   working through the redirect: existing locks, existing checkouts, hardcoded
   URLs.
2. **G1 (public pre-cliff PRs + operator gate)**: every known deploy flake whose
   lock pins `git+https://forge.anarch.diy/allod/profiles.git` re-points that
   input URL to `allod/archetypes` at the same revision — a URL-only
   flake.nix + lock rewrite; the rename preserves refs and objects, so the
   pinned rev stays reachable from the locked ref. The public template
   `allod/deploy` is itself such a consumer: a dedicated PR re-points its
   `profiles.url`, keeping the input *name* `profiles` until M3 renames it
   properly. Two other public PRs land in the same gate because they are
   runtime-reachable old-name consumers, not cosmetic sweep work:
   `allod/inventory` changes cold-clone repo lists and registry aliases so the
   framework checkout is `allod/archetypes` before the old URL dies, and
   `allod/nexus` moves deploy-flake defaults and rotation lock-bump/runbook
   steps to `allod/deploy` before a fresh `$HOME/work/allod/profiles` checkout
   can become the definitions-only repo. These changes are valid pre-split:
   the existing deploy template already re-exports `nixosConfigurations` and
   `vmFacts`. Operator deploy forks re-point likewise, existing local
   checkouts of the renamed repo re-point their remotes (`git remote set-url`)
   or get recloned at the `allod/archetypes` checkout path, and the operator
   confirms no live consumer still fetches through the redirect. This gate
   exists because step 3 is the point of no return for the redirect.
3. **M1 (human creates, agent populates)**: create the empty `allod/profiles`
   repo — this permanently ends the redirect; old-URL cold fetches now resolve
   to the new repo, whose fresh history contains none of the framework's revs.
   Agent populates it by PR against the new repo.
4. **M2**: the `archetypes` seam-flip PR, locking the populated
   `allod/profiles` as its `profiles` input. In this PR, every other input rev
   in `flake.lock` stays untouched, so the composition-parity comparison in
   Acceptance Tests is meaningful.
5. **M3**: the `allod/deploy` population PR. Depends on M2 (it pins the
   post-flip framework).
6. **M4**: the `secrets` template cleanup PR. Depends on M2 being merged (the
   framework no longer reads the removed exports). Order relative to M3 is
   free.
7. **M5**: final sweep PRs — add the new definitions repo checkout where useful,
   clean up remaining aliases/docs, regenerate generated files, then `memory`
   last, carrying the closing keyword. No cold-fetch or provisioning default
   may still depend on the old `allod/profiles` framework meaning at this point;
   those were G1 blockers.

Transitional states are harmless by construction: between M1 and M2 the example
definitions exist both in the framework's `hosts/` tree and in the new repo,
but nothing consumes the new repo until M2 locks it. Between M2 and M4 the
secrets template exports attrs nobody reads.

PR map: G1a `allod/deploy` (URL re-point), G1b `allod/inventory`
(framework checkout re-point), G1c `allod/nexus` (deploy-flake defaults and
rotation runbook re-point), M1 `allod/profiles`, M2 `allod/archetypes`, M3
`allod/deploy`, M4 `allod/secrets`, M5a `allod/inventory` (new definitions
checkout + final registry cleanup), M5b `allod/nexus` (remaining non-cliff
prose/tests only if needed), M5c `allod/memory`. Ten PRs, two human forge
operations (M0, M1 creation), one operator gate (G1).

## Risk Assessment

Overall residual risk: R3 High — cross-repo interfaces and sequencing, a
rename with an irreversible redirect cliff, and provisioning-adjacent pointer
changes. Per milestone:

| Milestone | Risk | Reason | Human scrutiny |
|---|---|---|---|
| M0 rename | R2 Medium | Forge-level and reversible until M1; the redirect keeps every consumer working. Blast radius is anything hardcoding the old URL, bridged by the redirect until G1/M1 complete. | Confirm the redirect serves: `git ls-remote` the old URL after renaming. Confirm M1 is *not* done in the same sitting. |
| G1a deploy URL re-point (`allod/deploy`) | R1 Low | URL-only input rewrite at the same locked rev; composed content identical by construction, proven by drvPath comparison. | The diff touches only the input URL and lock metadata; drvPaths byte-identical before/after. |
| G1b/G1c public runtime re-points (`inventory`, `nexus`) | R2 Medium | These are mechanical but provisioning-adjacent: they prevent M1 from turning cold clones, `DEPLOY_FLAKE` defaults, and rotation lock-bump instructions toward the new definitions-only repo. The deploy template already works pre-split, so rollback is straight revert before M1. | The inventory registry/spec diff and `nix flake check`; the nexus defaults/runbook/test diff and `nix flake check`; a grep showing no provisioning default or framework checkout still depends on the old `allod/profiles` meaning. |
| M1 new `allod/profiles` | R2 Medium | Additive fresh repo nobody consumes yet. The risk is entirely in the sequencing: creating it kills the redirect, so G1 must actually be complete, and the fresh history must carry no framework commits. | Verify G1 confirmations happened. Verify `git log` in the new repo shows only new commits. |
| M2 seam flip | R3 High | The composition path for every machine changes; wrong merge or default re-homing silently changes composed systems. Mitigated by drvPath parity on the whole example fleet plus the existing check suite. | The parity output; the diff hunks deleting `or {}` fallbacks (removed reads must be gone entirely, not defaulted); the re-homed `preferencesModule` and `profileData` reads. |
| M3 deploy template | R2 Medium | Adapts an existing template nothing operational builds from directly — but it pins the pattern operators fork, so a wrong follows shape here propagates into every future fork. | The three `follows` lines; the canary sabotage runs actually failing. |
| M4 secrets cleanup | R1 Low | Removes exports the framework no longer reads; straight revert. | The grep proving zero readers at M2's merged rev. |
| M5 sweep | R2 Medium | Mostly mechanical, but it still changes checkout inventory and public docs after the new definitions repo exists. The redirect-cliff defaults already moved in G1; M5 must not be the first time provisioning stops using the old framework path. | The inventory registry check and `vm-specs.json` regen; grep review of remaining `allod/profiles` hits as either the new definitions repo or the recorded History note. |

Rename costs, recorded up front (reference hygiene):

- All pre-rename commits remain fetchable at the `allod/archetypes` URL. No
  history rewrites anywhere; renames only.
- Historical deploy locks that pin `allod/profiles` revisions stop resolving
  cold once M1 claims the name (the pinned revs are not in the new repo's
  history). Cold re-evaluation of such a lock requires overriding the input
  URL: `--override-input <name> "git+https://forge.anarch.diy/allod/archetypes.git?rev=<rev>"`.
  The `archetypes` README records this mapping in a short History note — the
  one place archaeology will look.

## Interface Contracts

1. **The `profiles` input contract** — what any profiles repo (the public
   example repo and every operator repo alike) must export:

   ```nix
   lib.profileDefinitions = {
     <archetype> = {                  # dev | privacy | hypervisor
       <definition-name> = {
         override ? false;            # consumed only by composeProfileDefinitions layering
         nixosModules ? [ <module> ... ];
         homeModules  ? [ <module> ... ];
       };
     };
   };
   lib.profileData = {                # optional; absent machine keys are legal
     <machine-name> = { <builder-arg overrides> };
   };
   homeModules.preferences = <home-manager module>;
   ```

   `profileDefinitions` and `profileData` carry today's semantics unchanged,
   re-homed: definitions normalize through the framework's existing
   `normalizeDefinition` (unknown extra keys tolerated, as today). The
   framework's existing unknown-archetype assertion is already correct and must
   be left in its current direction:
   `unknownProfileDefinitionArchetypes = subtractLists profileArchetypes
   allProfileDefinitionArchetypes` evaluates to
   `declaredProfileDefinitionArchetypes - profileArchetypes` (nixpkgs
   `subtractLists e list` removes `e`'s elements from `list`), so an unknown
   archetype such as `service` is flagged and the `machineConfigurations`
   assert throws. M2's job is to add regression coverage, not to change the
   subtraction — reversing it would silently stop rejecting unknown archetypes,
   inverting a fail-loud guard. Because that binding runs only over the real
   `profileDefinitionLayers`, M2 must lift the unknown-archetype computation
   into a helper that accepts injected layers so the M2 sabotage case can drive
   it. A machine selecting a missing definition remains an eval-time throw, and
   `profileData.<machine>` merges into builder args exactly as
   `secrets.lib.profileData.<machine>` does today.
   `homeModules.preferences` is the module today exported by the secrets
   template.

2. **`archetypes` exports** (new in M2):

   - `profilesSource` (top-level output): the `outPath` string of the
     `profiles` input the flake actually composed with. Because flake input
     overrides rewrite the child's inputs, a deploy flake that redirects
     `archetypes/profiles` sees its own store path here; a deploy whose
     redirect was lost sees the framework's own locked public path instead.
   - `lib.composeProfileDefinitions { base, overlay }`: returns the merged
     definitions attrset. An overlay definition whose (archetype, name) exists
     in `base` without `override = true` is an eval error — today's
     public/private collision rule, framework-owned and exported so a layering
     profiles repo (one that imports the public examples and adds its own
     machines) reuses it instead of reimplementing it. The old
     secrets-vs-inventory duplicate check retires with the dual private
     sub-layers; it has no analog in a two-argument compose.
   - `lib.composedLayerCheck { pkgs, expectedProfiles }`: returns a derivation
     (runCommand, matching the existing check style) that fails iff
     `profilesSource != expectedProfiles.outPath`, with a message naming both
     store paths and the likely cause (a missing or dropped
     `archetypes.inputs.profiles.follows` line in the calling deploy flake).
     Building it is the generic canary.

   Removed in M2, outright — no `or {}` residue, so any stale layout fails
   loud at eval: the `secrets.lib.profileDefinitions` and
   `inventory.lib.profileDefinitions` layer reads, the
   `secrets.lib.profileData` read (becomes `profiles.lib.profileData.<name>
   or {}` — the `or {}` on the *machine key* stays; absent per-machine data is
   legal, as today), the `mkDevVm` default `preferencesModule ?
   secrets.homeModules.preferences` and `mkHypervisor`'s hardcoded
   `secrets.homeModules.preferences` import (both become the `profiles`
   input's module), and the in-tree `publicProfileDefinitions` attrset plus
   the three-layer `mergeProfileDefinitionLayers` machinery (survives only as
   the exported two-argument compose helper).

   Unchanged surfaces: `vmFacts` shape and derivation, `nixosConfigurations`
   naming (still composed in `archetypes` from its own locked inputs — the
   framework remains its own buildable example composition), builder
   semantics beyond the re-homed reads, the archetype set, the installer
   configuration, and every identity/credential read from `secrets`
   (identity is charter; only behavior moves).

3. **The `allod/deploy` template shape** — the thin adapter. The pre-split
   template (allod/deploy#1) already pins this shape minus the definitions
   input, with the framework input still named `profiles`; M3 transforms it
   into:

   ```nix
   inputs = {
     archetypes.url = "git+https://forge.anarch.diy/allod/archetypes.git";
     profiles.url   = "git+https://forge.anarch.diy/allod/profiles.git";
     secrets.url    = "git+https://forge.anarch.diy/allod/secrets.git";
     inventory.url  = "git+https://forge.anarch.diy/allod/inventory.git";
     archetypes.inputs.profiles.follows  = "profiles";
     archetypes.inputs.secrets.follows   = "secrets";
     archetypes.inputs.inventory.follows = "inventory";
   };
   outputs = { self, archetypes, profiles, ... }: {
     nixosConfigurations = archetypes.nixosConfigurations;
     vmFacts = archetypes.vmFacts;   # provisioning reads <flake>#vmFacts (allod/nexus#6)
     checks.<system>.composed-layer =
       archetypes.lib.composedLayerCheck { pkgs = ...; expectedProfiles = profiles; };
   };
   ```

   An operator fork changes exactly three input URLs (`profiles`, `secrets`,
   `inventory`) and nothing else — that is the "nothing else differs" charter
   both READMEs promise, now enforceable by diff against the template. The
   framework repos `vm` and `nexus` are not redirected; they arrive through
   `archetypes`' own lock and move with `nix flake update archetypes`. The
   template adds no policy beyond the canary check.

4. **New `allod/profiles` repo layout**: `hosts/<archetype>/<name>/` module
   files exactly as they exist in the framework today (`hosts/dev/allod-dev/
   {configuration,home}.nix`; `privacy-1` and `nexus` remain empty
   definitions), `modules/preferences.nix` (moved from the secrets template),
   and a `flake.nix` exporting contract 1 as a literal attrset — same
   explicitness as today's `publicProfileDefinitions`, no directory-derived
   magic. It needs no inputs beyond `nixpkgs` for its shape check; it must
   never grow inputs on `secrets` or `inventory`.

5. **Registry and pointer contract after G1/M5**: before M1, the inventory
   registry and machine repo lists stop using `allod/profiles` to mean the
   framework: the framework checkout is the full alias `allod/archetypes`, and
   the deploy composition root remains the full checkout `allod/deploy` for VM
   repo lists. Script defaults use bare aliases, so the registry also needs
   bare `deploy`, `secrets`, and `inventory` entries. The registry check's
   required-alias list becomes `deploy secrets inventory` in that pre-cliff
   inventory PR because the scripts that motivated requiring `profiles` now
   resolve the composition root instead. After M1/M3, M5 may add/keep
   `allod/profiles` as the new definitions repo checkout for humans who edit
   examples, but no provisioning default may rely on it for framework or
   deploy-flake behavior.

   `nexus` deploy defaults move in G1: `DEPLOY_FLAKE` defaults and the
   `/template/profiles` sentinel reference the deploy checkout
   (`$HOME/work/allod/deploy`, `/template/deploy`). Rotation commands that tell
   the operator to bump a flake lock (`forge-ssh-key`, `vm-ssh-host-key`,
   `nexus-host-key`, and `rotate-token`) must point at the deploy checkout too,
   with variable names renamed if keeping `MACHINE_PROFILES` /
   `PROFILES_CHECKOUT` would obscure the meaning. `assert_clean` should guard
   the checkout the command actually asks the operator to edit.

   Do not repoint optional profile hook lookup to deploy: in
   `bootstrap-vm-from-host.sh`, `${...}/scripts/hooks/<vm>.sh` is a
   definitions-profile concern. It should either keep resolving the
   profiles-definitions checkout (possibly under a clearer variable such as
   `PROFILE_HOOKS_CHECKOUT`) or be deleted if no public/private hook contract is
   intended; it must not silently look under the deploy template. `allod-dev`'s
   machine `repos` list has `allod/archetypes` before M1 and may add
   `allod/profiles` after the examples repo exists; `vm-specs.json` is
   regenerated (`nix eval .#lib.vmSpecsJson --raw | jq -S .`) in the same PR
   that changes the repo list so its check stays green.

## Agent Gates

- **Forge repo operations are human-only**: the M0 rename, the M1 repo
  creation, new-repo settings (description, branch protection), and all PR
  merges. Blocks every milestone start; each PR waits on its human merge.
- **G1's private half is an operator confirmation.** The public half — the
  `allod/deploy`, `allod/inventory`, and `allod/nexus` pre-cliff PRs — is
  agent work, but the agent cannot enumerate deploy flakes or checkouts outside
  the public org, so it cannot verify that all old-URL consumers have
  re-pointed. The human confirms G1 before creating the new repo. Blocks M1.
- **Full-fleet parity is operator-side.** This plan's parity evidence covers
  the public example fleet only; parity for an operator's real fleet belongs
  to the companion operator-side plan.

## Acceptance Tests

M0/G1 (human-observable, recorded in the tracking issue):

```sh
# after the rename, before M1: the redirect serves the framework
git ls-remote https://forge.anarch.diy/allod/profiles.git   # same heads as allod/archetypes
```

G1a re-point PR (`allod/deploy`):

```sh
# URL-only: the composed content is provably unchanged
nix eval .#vmFacts --json >/dev/null
for m in allod-dev privacy-1 nexus installer; do
  nix eval .#nixosConfigurations.$m.config.system.build.toplevel.drvPath
done   # identical to the pre-re-point values
git diff master -- flake.nix   # touches only the profiles input URL
```

G1 runtime re-point PRs (`allod/inventory`, `allod/nexus`) — before M1:

```sh
# inventory: cold clones no longer fetch the framework through allod/profiles
nix flake check
jq -e '."allod-dev".repos | index("allod/archetypes")' scripts/vm-specs.json
jq -e '."allod-dev".repos | index("allod/profiles") | not' scripts/vm-specs.json
jq -e '.repositories["allod/archetypes"].remote == "allod/archetypes"' scripts/repositories.json
jq -e '.repositories.deploy.remote == "allod/deploy"' scripts/repositories.json
jq -e '.repositories.secrets.remote == "allod/secrets"' scripts/repositories.json
jq -e '.repositories.inventory.remote == "allod/inventory"' scripts/repositories.json

# nexus: deploy-flake defaults and rotation lock-bump steps use deploy before
# the profiles checkout can become definitions-only. These patterns match only
# literal old-name targets. Do NOT grep for `cd ${MACHINE_PROFILES}` /
# `cd ${PROFILES_CHECKOUT}`: §5 permits keeping those variable names while
# repointing their default to the deploy checkout, so their presence is not a
# failure — the resolved default is verified by the diff and `nix flake check`,
# not by a variable name. (The earlier `cd \$\{MACHINE_PROFILES\}` alternative
# was also double-escaped and never matched.)
nix flake check
if git grep -nE 'DEPLOY_FLAKE=.*allod/profiles|--flake ~/work/allod/profiles|cd ~/work/allod/profiles|Update profiles flake lock' scripts nix docs tests; then
  echo "old deploy-flake/lock-bump literal target still present"
  exit 1
fi

# Catch-all: every remaining allod/profiles hit in nexus before M1 must be
# optional profile hook lookup, a deploy-flake default doc now reading
# allod/deploy (docs/provisioning-scripts.md's DEPLOY_FLAKE / deployFlake
# defaults and the ~/work/allod/profiles smoke-test line are here, missed by the
# literal patterns above), or explicit historical/new-definitions prose. Review
# each.
git grep -n 'allod/profiles' scripts nix docs tests || true
```

M1 (new `allod/profiles`):

```sh
nix flake check          # shape check: profileDefinitions/profileData structure,
                         # list/bool field types; NOT archetype-name validity
                         # (that assertion stays in archetypes, the fact's owner)
nix eval .#lib.profileDefinitions --json | jq 'keys'   # dev, hypervisor, privacy
git log --oneline | wc -l                              # small; fresh history only
```

M2 (`allod/archetypes`), run in the PR worktree:

```sh
# Parity: identical drvPaths for the whole example fleet, or every delta
# explained exactly. Run the same eval at the base rev first; all non-profiles
# input revs in flake.lock are unchanged by this PR, so the comparison is
# meaningful. (Module content moves between store paths, which is invisible to
# the drv unless a path leaks into config — the parity run is what proves it
# didn't.)
for m in allod-dev privacy-1 nexus installer; do
  nix eval .#nixosConfigurations.$m.config.system.build.toplevel.drvPath
done

nix flake check          # full existing suite + any new check; if memory-tight,
                         # fall back to per-config toplevel evals + nix build
                         # per check, same coverage

# The removed reads are gone entirely:
git grep -nE 'secrets\.(lib\.profile|homeModules)' flake.nix   # empty
git grep -n 'inventory.lib.profileDefinitions'                 # empty
git grep -n 'publicProfileDefinitions'                         # empty
test ! -e hosts            # examples moved out, home-shared relocated to modules/

# The profile-definition contract check includes a sabotage case that feeds an
# unsupported archetype key (for example `service`) through the unknown-archetype
# assertion and must fail with its "unknown profile definition archetype(s)"
# message. It must fail *there*, NOT via the existing missing-selected-definition
# throw and NOT via a bare "attribute 'service' missing" from
# mergeProfileDefinitionLayers (which genAttrs-es only the known archetypes and
# silently drops unknown keys — a false green). This confirms the existing
# subtraction direction stays correct after the layers are made injectable.
nix build .#checks.x86_64-linux.profile-definition-contracts
```

M3 (`allod/deploy`):

```sh
nix flake check                                    # canary green on the template

# Sabotage 1 — lost redirect: archetypes composes a different profiles than the
# deploy pins. Must FAIL with both store paths in the message.
nix build .#checks.x86_64-linux.composed-layer \
  --override-input archetypes/profiles "git+https://forge.anarch.diy/allod/profiles.git?rev=<any-older-rev>" \
  && echo "SABOTAGE 1 NOT CAUGHT" && exit 1

# Sabotage 2 — definitions dropped: both deploy and framework see an empty
# profiles layer (tests/empty-profiles fixture flake exporting empty contract
# attrs). Machine eval must THROW "selects missing … profile definition".
nix eval .#nixosConfigurations.allod-dev.config.system.build.toplevel.drvPath \
  --override-input profiles path:./tests/empty-profiles \
  --override-input archetypes/profiles path:./tests/empty-profiles \
  && echo "SABOTAGE 2 NOT CAUGHT" && exit 1

# Stranger test: a fresh clone of the template alone evaluates and checks green
# with no credentials beyond repo read access.
git clone <deploy-url> /tmp/deploy-fresh && cd /tmp/deploy-fresh && nix flake check
```

The canary counts as validated only after both sabotage runs demonstrably fail
— a check that cannot be shown to fail on sabotaged input does not count.

M4 (`allod/secrets`):

```sh
git grep -nE 'profileDefinitions|profileData|preferences'   # empty
nix flake check                                             # template checks green
# In an archetypes checkout: the merged framework is green against the cleaned
# template without a lock commit:
nix flake check --override-input secrets "git+https://forge.anarch.diy/allod/secrets.git?rev=<M4-rev>"
```

M5 (sweep):

```sh
# inventory: registry + specs stay consistent
nix flake check          # repository-registry + vm-specs-json

# nexus: script suite still green after pointer updates
nix flake check

# Nothing left pointing at the old framework name anywhere public, except the
# recorded History note and intentional references to the NEW profiles repo:
for r in archetypes profiles deploy secrets inventory nexus vm tools memory strategy; do
  git -C <checkout-of-$r> grep -n 'allod/profiles' || true
done   # review every hit; each is either the History note or means the new repo
```

## Rollback Plan

- **M0**: rename back. Safe and complete until M1 creates the replacement
  repo; after M1, rolling back the rename first requires deleting or renaming
  away the new repo (destroying its fresh history) — which is why M0 and M1
  are separate human sessions separated by G1.
- **G1a deploy URL re-point**: revert the PR; the redirect still serves until
  M1, so the old URL keeps resolving.
- **G1b/G1c public runtime re-points**: before M1, straight revert. After M1,
  do not roll these back unless the replacement `allod/profiles` repo is also
  removed or all old-name consumers are otherwise proven unreachable; reverting
  them after redirect death makes cold clones and deploy defaults target the
  definitions-only repo.
- **M1**: delete the new repo. Clean until M2 locks it; after M2, revert M2
  first.
- **M2**: `git revert` of the seam PR restores the in-tree definitions and the
  `secrets`/`inventory` reads; the new profiles repo goes dormant but stays
  valid. No lock surgery needed — the PR touched only the `profiles` node.
- **M3**: revert; the template returns to the pre-split adapter shape
  (allod/deploy#1 plus the G1 re-point). Nothing operational builds from it.
- **M4**: revert restores dead-but-harmless exports. Only meaningful if M2 was
  also reverted; revert M4 before M2 in that case so the restored framework
  reads find their attrs.
- **M5**: straight reverts per repo. The inventory revert must re-regenerate
  `vm-specs.json` (the check forces this).
- **Partial states**: every PR is single-repo and atomic; the cross-repo
  in-between states are exactly the transitional states in Sequencing, all
  harmless by construction because consumers move only when their own PR
  lands.
