# Private-First Stack Migration — Agent Prompt

You have private `vnprc/*` access. Read `allod/memory/memory.md` and relevant
topic files before starting.

## Goal

Public `allod/{profiles,nexus,vm}` become the only code repos. Private
`vnprc/{secrets,inventory}` keep all real identity and machine data.
`vnprc/{profiles,nexus,vm}` get deleted with zero feature loss. Onboarding for
a stranger stays simple: clone the public repos, init identity into local
secrets/inventory clones, provision.

## Approach (decided)

Develop in `vnprc/*`, where the working stack and real data live; validate
there; reconcile code into the public repos at the end. The public code repos
are single-commit snapshots and not yet usable (ssh-only flake inputs, missing
host-key state, fake credentials) — do not develop in them. Follow workspace
conventions: dev plan and tracking issue before implementation PRs.

## Facts from the 2026-07-10 public-side review (do not re-derive)

- Builds bind data at lock time (`profiles` flake inputs pin
  secrets/inventory) while nexus scripts read runtime data from
  registry-resolved local checkouts (`nix eval path:$IDENTITY_CONFIG#...`).
  Proposal on the table: provision/rebuild/installer builds always pass
  `--override-input secrets path:<checkout> --override-input inventory
  path:<checkout>` so one profiles repo serves any data source. Validate
  against private practice; adopt or refute.
- Public `profiles` fetches nexus and inventory over `git+ssh://`;
  reconciliation must land `git+https://` for credentialless fetch.
- `forge_key = null` means "privacy VM" across provision, bootstrap, and
  verify; a credentialless public dev VM needs a repo-list discriminator.
- Schema split: `bootstrap-vm-from-host.sh` and nexus fixtures read
  `forge-ssh-keys.json[].secret`; public secrets ships `secret_path`.
  Determine which side is current; fix once.
- `machine-host-keys.json` feeds both nexus known_hosts pinning
  (`StrictHostKeyChecking=yes`) and profiles' `credential-profiles` check.
  `vm-ssh-host-key init` already generates and commits per-VM host-key state
  into a secrets checkout — that is the intended public onboarding step, not
  new runtime machinery.
- The public-side plan and its review fixes
  (`allod/strategy/dev-plans/allod-dev-public-profile.md`, commits
  `7d4c04c..3419c4f`) predate the private-first decision. Reference, not
  gospel; it gets rewritten from your results.

## Task

1. Diff `vnprc/{nexus,vm,profiles}` against `allod/{nexus,vm,profiles}`.
   Identify every checkout by `git remote -v`, never by path — both sides can
   occupy the same local paths. Classify each delta: general code (stays in
   code repos, syncs public), identity or machine specifics (migrate into
   secrets/inventory as data), or dead (drop). Look hard at host hardware
   config, `devVmOverrides`, `hosts/<vm>/`, `scripts/hooks/<vm>.sh`, service
   VM types, and post-snapshot script fixes. The private definition of the
   running `allod-dev` VM is the model for the stock public template values.
2. Write the migration dev plan on the private side and execute it in
   `vnprc/*`: move identity and machine specifics into secrets/inventory, make
   the code repos data-agnostic, and converge them toward byte-identical with
   their public counterparts.
3. Validate with fixtures and flake checks only; real provisioning and
   rebuilds are host-only — hand the human exact commands and expected
   results.
4. Produce the reconciliation map: ordered commits/files to sync into the
   public repos, which public-plan sections your work supersedes, and the
   delete-preconditions for `vnprc/{nexus,vm,profiles}`.

## Rules

- You hold private access: never push public `allod/*` repos or open public
  PRs. Deliverables live private-side; the human relays.
- Anything that may be relayed into public text: shapes, paths, field names
  only — no keys, tokens, addresses, personal identifiers.
- No rekeying, no Forgejo mutation, no host-only commands.
