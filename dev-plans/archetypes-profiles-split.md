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
- `allod/deploy` adaptation (M3): rename the framework input `profiles` ->
  `archetypes`, add the `profiles` definitions input with its
  follows-redirect, add the composed-layer canary as a flake check plus a
  sabotage fixture, and update the README. (The G1-time URL-only re-point is
  a separate, earlier PR — see Sequencing.)
- `secrets` template cleanup (M4): remove `lib.profileDefinitions`,
  `lib.profileData`, `homeModules.preferences`, and `modules/preferences.nix`;
  README ownership touch-up.
- Sweep (M5): `inventory` registry aliases, machine `repos` lists,
  `vm-specs.json` regen; `nexus` provisioning pointer defaults and runbook
  prose; `memory` topic files; grep-driven cleanup of remaining `allod/profiles`
  references across public repos.

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
2. **G1 (one public PR + operator gate)**: every known deploy flake whose
   lock pins `git+https://forge.anarch.diy/allod/profiles.git` re-points that
   input URL to `allod/archetypes` at the same revision — a URL-only
   flake.nix + lock rewrite; the rename preserves refs and objects, so the
   pinned rev stays reachable from the locked ref. The public template
   `allod/deploy` is itself such a consumer: a dedicated PR re-points its
   `profiles.url`, keeping the input *name* `profiles` until M3 renames it
   properly. Operator deploy forks re-point likewise, existing local
   checkouts of the renamed repo re-point their remotes
   (`git remote set-url`) or get recloned, and the operator confirms no live
   consumer still fetches through the redirect. This gate exists because
   step 3 is the point of no return for the redirect.
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
7. **M5**: sweep PRs — `inventory`, `nexus`, then `memory` last, carrying the
   closing keyword.

Transitional states are harmless by construction: between M1 and M2 the example
definitions exist both in the framework's `hosts/` tree and in the new repo,
but nothing consumes the new repo until M2 locks it. Between M2 and M4 the
secrets template exports attrs nobody reads.

PR map: G1 `allod/deploy` (URL re-point), M1 `allod/profiles`, M2
`allod/archetypes`, M3 `allod/deploy`, M4 `allod/secrets`, M5a
`allod/inventory`, M5b `allod/nexus`, M5c `allod/memory`. Eight PRs, two
human forge operations (M0, M1 creation), one operator gate (G1).

## Risk Assessment

Overall residual risk: R3 High — cross-repo interfaces and sequencing, a
rename with an irreversible redirect cliff, and provisioning-adjacent pointer
changes. Per milestone:

| Milestone | Risk | Reason | Human scrutiny |
|---|---|---|---|
| M0 rename | R2 Medium | Forge-level and reversible until M1; the redirect keeps every consumer working. Blast radius is anything hardcoding the old URL, bridged by the redirect until G1/M1 complete. | Confirm the redirect serves: `git ls-remote` the old URL after renaming. Confirm M1 is *not* done in the same sitting. |
| G1 re-point (`allod/deploy`) | R1 Low | URL-only input rewrite at the same locked rev; composed content identical by construction, proven by drvPath comparison. | The diff touches only the input URL and lock metadata; drvPaths byte-identical before/after. |
| M1 new `allod/profiles` | R2 Medium | Additive fresh repo nobody consumes yet. The risk is entirely in the sequencing: creating it kills the redirect, so G1 must actually be complete, and the fresh history must carry no framework commits. | Verify G1 confirmations happened. Verify `git log` in the new repo shows only new commits. |
| M2 seam flip | R3 High | The composition path for every machine changes; wrong merge or default re-homing silently changes composed systems. Mitigated by drvPath parity on the whole example fleet plus the existing check suite. | The parity output; the diff hunks deleting `or {}` fallbacks (removed reads must be gone entirely, not defaulted); the re-homed `preferencesModule` and `profileData` reads. |
| M3 deploy template | R2 Medium | Adapts an existing template nothing operational builds from directly — but it pins the pattern operators fork, so a wrong follows shape here propagates into every future fork. | The three `follows` lines; the canary sabotage runs actually failing. |
| M4 secrets cleanup | R1 Low | Removes exports the framework no longer reads; straight revert. | The grep proving zero readers at M2's merged rev. |
| M5 sweep | R2 Medium | Mechanical pointers, but they touch provisioning defaults (`DEPLOY_FLAKE`, registry aliases, rotation-runbook checkout resolution); a wrong default strands provisioning on a host without env injection. | The `nexus` defaults diff; the inventory registry check and `vm-specs.json` regen. |

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
   `normalizeDefinition` (unknown extra keys tolerated, as today), archetype
   names are validated by the framework's existing unknown-archetype
   assertion at consumption, a machine selecting a missing definition remains
   an eval-time throw, and `profileData.<machine>` merges into builder args
   exactly as `secrets.lib.profileData.<machine>` does today.
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

5. **Registry and pointer contract after the sweep (M5)**: the inventory
   registry gains `archetypes` and `deploy` entries; the bare `profiles`
   alias re-points to the new definitions repo; the registry check's
   required-alias list becomes `deploy secrets inventory` (the scripts that
   motivated requiring `profiles` now resolve the composition root instead).
   `nexus` script defaults move accordingly: `DEPLOY_FLAKE` defaults and the
   `/template/profiles` sentinel reference the deploy checkout
   (`$HOME/work/allod/deploy`, `/template/deploy`); `MACHINE_PROFILES` /
   `PROFILES_CHECKOUT` in `bootstrap-vm-from-host.sh`, `forge-ssh-key`, and
   `nexus-host-key` resolve the `deploy` alias — post-split, the checkout
   whose lock pins `secrets` (which is what those scripts bump and
   `assert_clean`) is the deploy composition root, not the definitions repo.
   Keeping the variable names while re-pointing their targets is the default;
   renaming them for honesty is reviewer's latitude. `allod-dev`'s machine
   `repos` list adds `allod/archetypes` and keeps `allod/profiles` (now the
   examples repo); `vm-specs.json` is regenerated (`nix eval
   .#lib.vmSpecsJson --raw | jq -S .`) in the same PR so its check stays
   green.

## Agent Gates

- **Forge repo operations are human-only**: the M0 rename, the M1 repo
  creation, new-repo settings (description, branch protection), and all PR
  merges. Blocks every milestone start; each PR waits on its human merge.
- **G1's private half is an operator confirmation.** The public half — the
  `allod/deploy` URL re-point PR — is agent work, but the agent cannot
  enumerate deploy flakes or checkouts outside the public org, so it cannot
  verify that all old-URL consumers have re-pointed. The human confirms G1
  before creating the new repo. Blocks M1.
- **Full-fleet parity is operator-side.** This plan's parity evidence covers
  the public example fleet only; parity for an operator's real fleet belongs
  to the companion operator-side plan.

## Acceptance Tests

M0/G1 (human-observable, recorded in the tracking issue):

```sh
# after the rename, before M1: the redirect serves the framework
git ls-remote https://forge.anarch.diy/allod/profiles.git   # same heads as allod/archetypes
```

G1 re-point PR (`allod/deploy`):

```sh
# URL-only: the composed content is provably unchanged
nix eval .#vmFacts --json >/dev/null
for m in allod-dev privacy-1 nexus installer; do
  nix eval .#nixosConfigurations.$m.config.system.build.toplevel.drvPath
done   # identical to the pre-re-point values
git diff master -- flake.nix   # touches only the profiles input URL
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
nix flake check          # repository-registry (new required list) + vm-specs-json

# nexus: script suite still green after pointer updates
nix flake check

# Nothing left pointing at the old name anywhere public, except the recorded
# History note and intentional references to the NEW profiles repo:
for r in archetypes profiles deploy secrets inventory nexus vm tools memory strategy; do
  git -C <checkout-of-$r> grep -n 'allod/profiles' || true
done   # review every hit; each is either the History note or means the new repo
```

## Rollback Plan

- **M0**: rename back. Safe and complete until M1 creates the replacement
  repo; after M1, rolling back the rename first requires deleting or renaming
  away the new repo (destroying its fresh history) — which is why M0 and M1
  are separate human sessions separated by G1.
- **G1 re-point**: revert the PR; the redirect still serves until M1, so the
  old URL keeps resolving.
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
