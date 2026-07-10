# Public/Private Divergence Audit — Agent Prompt

You are an infrastructure archaeologist with something the last reviewer did not
have: private `vnprc/*` access. The previous agent reviewed the public template
blind — it probed `vnprc/*` and got 401s, its `allod_vm` key authenticates as
`allod-agent` with no private read, and every claim it made about the private
side is inference. You can see both sides. Your job is to map the real
divergence so the transition plan stops being guesswork.

Read `allod/memory/memory.md` and its relevant topic files before starting.

## Why This Audit Exists

The human's actual goal, stated after review pass 1:

> Complete the transition from private profiles, nexus, and vm repos to the
> public ones. I should be able to delete my private nexus and vm forks and
> still use the full stack with no compromises. I also want a clean and simple
> as possible onboarding path.

Target architecture (pending your audit): **public code, private data.** The
public `allod/{profiles,nexus,vm}` repos become the only code repos; private
`vnprc` secrets and inventory keep carrying real identity; the private code
forks get deleted.

## Conversation State

1. **Review pass 1** (2026-07-10, public-only agent) of
   `allod/strategy/dev-plans/allod-dev-public-profile.md` (tracking issue
   `allod/profiles#2`) found 3 BLOCKERs + 6 GAPs, all original-plan defects,
   all fixed in plan text: commits `7d4c04c`, `9378e1a`, `a6ca651`, `21d1b39`,
   `c6d8ef3`, `e070316`, `a90037a`, `2918f51`, `2597f81`, prompt update
   `3419c4f`, on `allod/strategy` master. Highlights:
   - Public `profiles` fetches its `nexus`/`inventory` flake inputs over
     `git+ssh://` — credentialless builds are impossible today.
   - Host-key trust chain: `bootstrap-vm-from-host.sh`/`verify-vm-from-host`
     pin `KNOWN_HOSTS_VMS` from `machine-host-keys.json` with
     `StrictHostKeyChecking=yes` and die without an entry; profiles'
     `credential-profiles` check requires `credentials."<vm>-host"` from the
     same file.
   - Operator SSH trust is baked at build time from synthetic
     `hostPublicKeys` (`vm/modules/qemu-guest.nix` authorized_keys, installer
     root keys).
   - `forge_key = null` means "privacy VM" in provision/bootstrap/verify
     gating.
   - Public schema split: `bootstrap-vm-from-host.sh` reads `.secret` from
     `forge-ssh-keys.json`; public `allod/secrets` ships `secret_path`.
   - All eight `allod/*` repos verified anonymously clonable (HTTP 200).
2. **Strategy pivot**: the zero-edit demo goal was judged to buy little and
   cost real features (rotation state, agenix path, declarative trust all go
   conditional or unexercised). Proposed replacement, pending your findings:
   - Provision/rebuild/installer builds always pass
     `--override-input secrets path:<checkout> --override-input inventory
     path:<checkout>`, resolved from the registry — the scripts already treat
     local checkouts as authoritative for runtime data
     (`nix eval path:$IDENTITY_CONFIG#...`) while builds consume locked
     upstream inputs, a latent split.
   - Onboarding: clone → `vm-ssh-host-key init allod-dev` (exists; writes and
     commits host-key state into the local secrets clone) → add the operator's
     host pubkey to local `identity.nix` → `provision-vm-from-host allod-dev`.
   - Stock public `allod-dev` keeps `self_rebuild = false`; `self_rebuild =
     true` is for deployments whose data repos are reachable from inside the
     VM.
   - If adopted: plan commits `9378e1a` (runtime host-key generation) and
     `a6ca651` (mutable authorized_keys injection + installer override) get
     superseded; the rest survives.
3. **Environment facts** established from inside the running `allod-dev` VM:
   hostname `allod-dev`, user `allod`, sole credential is the `allod_vm` SSH
   key (Forgejo account `allod-agent`, public-org access only), no
   netrc/token files anywhere, and the rendered `~/.ssh/config` still carries
   a stale `dev-1` block pointing at `192.0.2.10` with a nonexistent
   `~/.ssh/dev_1` identity file — live evidence of the `sshHosts` staleness
   flagged in review finding 8.

## Your Task

Map the divergence between `vnprc/{nexus,vm,profiles,secrets,inventory}` and
their `allod/*` counterparts, and validate or refute the proposed
architecture. Ground every claim in trees and history (`git remote -v`,
`git log`, file reads) — public and private checkouts may occupy identical
local paths, so identify every repo you read by its remote, not its
directory name. The public code repos are single-commit snapshots
("initial public release"); date them against private history.

Answer these, with evidence:

1. **Code divergence per repo.** For each of `nexus`, `vm`, `profiles`: what
   does private have that public lacks (or vice versa)? Look specifically for
   real host hardware config, `devVmOverrides` (public stubs it to `{}`),
   `scripts/hooks/<vm>.sh` post-provision hooks, service VM types, extra
   modules, and script fixes that postdate the snapshot. Classify each delta:
   upstream to public / becomes data in secrets-inventory / drop.
2. **Data binding today.** How does the private deployment point builds at
   private data — private `profiles` flake inputs aimed at `vnprc`
   secrets/inventory, `--override-input`, path inputs, something else? Does
   the proposed override-input-from-registry-checkouts match, improve, or
   fight current practice? Does it work for the hypervisor build path too?
3. **Schema currency.** For each shared contract file — `identity.nix` shape,
   `credentials.nix`, `machine-host-keys.json`, `forge-ssh-keys.json`
   (`.secret` vs `secret_path`), `forgejo-token-groups.json`,
   `repositories.json`, `vm-specs.json` (including `github_repos`) — which
   side is current? Which public snapshot is stale, and does that change any
   pass-1 review conclusion?
4. **The real allod-dev.** This VM exists and runs agents today. Where is it
   defined and how was it provisioned — private inventory entry, private
   secrets identity, which scripts? Its private definition is the working
   model for the public contract values; compare them (repo set, forge key
   posture, self_rebuild, credential wiring for the `allod_vm`/`allod-agent`
   setup).
5. **Cutover blockers.** Enumerate everything that still references
   `vnprc/{nexus,vm,profiles}`: flake inputs across private repos, the host's
   NixOS config, registry entries, scripts, docs. This becomes the
   delete-preconditions checklist.
6. **No-compromise check.** The bar is: delete the private code forks and lose
   nothing. List anything private-only that the public repos must absorb
   first, and anything that cannot be expressed as public-code-plus-private-
   data under the proposed architecture.

## Ground Rules

- You hold private access, so per the standing agent gate you must **not push
  public `allod/*` repos or open public PRs**. Deliver the report on the
  private side (a `vnprc` strategy checkout if one exists, otherwise a local
  file whose path you print); the human relays conclusions.
- Write conclusions so they are safe to relay into public plan text: shapes,
  paths, field names, commit subjects — never key material, tokens, real
  addresses, or personal identifiers.
- Read-only otherwise: no rekeying, no host-only commands, no Forgejo
  mutation.

## Deliverable

One markdown divergence report containing:

1. A per-repo divergence table with each delta classified (upstream / to-data
   / drop).
2. Evidence-backed answers to task questions 2–6.
3. A verdict on the proposed architecture — "public code + swappable data via
   override-input, onboarding via `vm-ssh-host-key init`, delete
   `vnprc/{nexus,vm,profiles}`" — sound as-is, sound with amendments (state
   them), or wrong (state why and propose the alternative).
4. A recommended cutover order: upstream commits → data migration → repoint →
   verify → delete.
5. The list of pass-1 plan commits to keep or supersede in the rewrite, so the
   public-side agent can execute it without re-deriving your findings.

The plan rewrite itself will be executed afterward by a public-only agent
working from this report.
