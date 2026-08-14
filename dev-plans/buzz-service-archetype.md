# Buzz Service Archetype

## Tracking Issue

`allod/archetypes` issue #15 — https://forge.anarch.diy/allod/archetypes/issues/15

Multi-PR work across `allod/archetypes`, `allod/profiles`, `allod/inventory`, and
`allod/secrets`. Earlier PRs carry `Refs allod/archetypes#15`; the final
`allod/archetypes` implementation PR (M3d below) carries the closing keyword.

This is the public half of a cross-boundary arc and a self-contained leaf: it
names no private machine, path, or key, and a public-only agent can execute and
validate all of it. Downstream deploy forks own concrete machines, addressing,
key material, backups, and rollout, and consume the Interface Contracts below.

## Goal

The framework composes small headless always-on service VMs that serve other
machines on the local network — `mkServiceVm` replaces the `flake.nix` stub —
with the buzz relay stack as the first consumer, proven by one synthetic example
machine and generated-artifact checks.

## Scope

In scope:

- `allod/archetypes` — archetype merge support for a `service` class
  (`profileArchetypes`, `builders.service = mkServiceVm`, the guard extensions
  that accept the new class); a shared service-class module (headless,
  always-on, fail-closed firewall with a per-machine source-scoped port
  allowlist, no forge write capability by default); a buzz package (relay +
  `buzz` CLI from github.com/block/buzz pinned at
  `2ce2d71cc38a9657eaf344c10e07f155b8a18615`, Apache-2.0); a buzz stack NixOS
  module composing native Postgres/Redis/MinIO services with the relay unit;
  checks, with platforms derived from `inventory.lib.supportedPlatforms`.
- `allod/profiles` — one synthetic example service definition
  (`profileDefinitions.service.service-1`) and its
  `hosts/service/service-1/configuration.nix`.
- `allod/inventory` — the synthetic `service-1` machine facts and the
  `scripts/vm-specs.json` regeneration.
- `allod/secrets` — the synthetic `service-1` identity, the new
  `lib.serviceIdentities` export, `vmUsernames` coverage, and the synthetic
  host-key material the `vmFacts` seam requires.
- `allod/archetypes` `modules/dev-home-shared.nix` — the pinned buzz CLI on
  every dev VM PATH; `mkHypervisor` gains the CLI in
  `home-manager.extraSpecialArgs`.

Out of scope, each with its owner:

- Concrete machines, addressing, key material, backups, deployment, and rollout
  sequencing — downstream deploy forks.
- `allod/deploy` — no change. The template re-exports
  `archetypes.nixosConfigurations` and picks up the new machine at its own
  routine lock advance.
- Hypervisor-side autostart policy for service VMs — host configuration, owned
  by the deployment.
- Any client tooling beyond packaging the upstream CLI — follow-up work when a
  consumer wants it.

## Milestones and Landing Order

Entry triggers and exit criteria gate each milestone; nothing starts on
calendar time.

### M1 — buzz package (`allod/archetypes`, one PR)

Entry: this plan accepted and issue #15 open.

- `packages/buzz/default.nix` builds the workspace's `buzz-relay` and
  `buzz-cli` crates from `fetchFromGitHub { owner = "block"; repo = "buzz";
  rev = "2ce2d71cc38a9657eaf344c10e07f155b8a18615"; hash = ...; }` — the pin
  lives in-tree so a routine `nix flake update` cannot move it.
- Built with `rustPlatform` from the flake's existing `nixpkgs-unstable`
  input: the pinned rev's `rust-toolchain.toml` requires rustc 1.95.0, and at
  the current locks the stable input carries 1.91.1 while the unstable input
  carries 1.97.0.
- Dependencies vendor through `importCargoLock` over a copy of the upstream
  `Cargo.lock` committed beside the package expression (avoids
  import-from-derivation); the lock's two git-sourced dependencies need pinned
  `outputHashes`.
- The upstream Apache-2.0 `LICENSE` is copied alongside the vendored lock
  file. No `NOTICE` file exists at the pinned rev; if a future pin adds one,
  copy it too.
- Flake outputs `packages.<system>.buzz` (both binaries) and
  `packages.<system>.buzz-cli` (a `bin/buzz`-only view of the same build, no
  second compile), keyed by `inventory.lib.supportedPlatforms` — never a
  literal system list (public memory `nix.md`, machine platform literals
  belong to inventory).

Exit: M1 acceptance tests pass; PR merged.

### M2 — service archetype and modules (`allod/archetypes`, one PR)

Entry: M1 merged.

- `mkServiceVm` replaces the stub; `profileArchetypes` gains `"service"`;
  `builders.service = mkServiceVm`; the identities merge consumes
  `secrets.lib.serviceIdentities` (Interface Contracts 3).
- `modules/service.nix` — the shared service-class module: headless always-on
  baseline plus the firewall allowlist option (Interface Contracts 4).
- `modules/buzz-stack.nix` — the buzz stack module (Interface Contracts 5),
  imported by the builder for every service VM and inert until a profile sets
  `services.buzz.enable`.
- `modules/dev-home-shared.nix` embeds `buzz-cli`; `mkHypervisor` passes
  `buzz-cli` through `home-manager.extraSpecialArgs` (Interface Contracts 7).
- New check `service-stack-contracts` drives the module assertions positively
  and negatively through synthetic per-platform evaluations — no inventory
  machine needed.
- No machine composes the class yet: the M2 diff must leave every existing
  example machine's `toplevel.drvPath` byte-identical.

Exit: M2 acceptance tests pass, including the drvPath parity run; PR merged.

### M3 — example machine and checks (four PRs)

Entry: M2 merged.

- **M3a `allod/secrets`**: `identity.nix` gains a `serviceVMs` set
  (`service-1` with username `service`) and an `sshHosts.service-1` entry; the
  flake exports `lib.serviceIdentities` and extends `vmUsernames`;
  `machine-host-keys.json` gains a `service-1` entry (synthetic active key,
  `staged = null`); the synthetic `secrets/vm-host-keys/service-1-ssh.age` and
  its `secrets.nix` recipients line land together. The `service-1-host`
  credential record auto-derives from `machine-host-keys.json` — no
  hand-written `credentials.nix` entry.
- **M3b `allod/inventory`**: `machines.service-1` (Interface Contracts 3),
  sized from the issue's evaluation numbers (the stack idles around 225MB:
  relay 34MB, Postgres 30MB, Redis 9MB, MinIO 152MB); regenerate
  `scripts/vm-specs.json` in the same PR.
- **M3c `allod/profiles`**: `hosts/service/service-1/configuration.nix`
  enabling the stack with synthetic values — relay port 3000 allowlisted to
  the inventory template's synthetic dev-machine address `192.0.2.10`, health
  and metrics listeners left closed, `environmentFile =
  "/var/lib/buzz/relay.env"` as the documented runtime path a deploy fork
  points at its own key delivery — registered as
  `profileDefinitions.service.service-1`; update the flake comment that
  enumerates valid archetype names.
- **M3d `allod/archetypes`**: one commit advances the `secrets`, `inventory`,
  and `profiles` locks together; the `service-machine-artifacts` check lands;
  the `credential-profiles` check's VM filter extends to include `"service"`;
  README touch; the PR carries the closing keyword.

Landing constraint — the one sequencing hazard: after M2, the framework's
`machines keys must exactly match identity keys` assertion makes a partial lock
advance red. Bumping `inventory` without `secrets` composes a machine without
an identity; the reverse composes an identity without a machine. M3a/b/c may
merge in any order (template masters are inert until locked), but between the
first of them merging and M3d merging, no routine `allod/archetypes`
lock-update commit may advance `inventory`, `secrets`, or `profiles`
individually. M3d advances all three in one commit, validated pre-merge with
`--override-input` against the branch revs (public memory `nix.md`, validating
an unmerged cross-repo change).

Exit: M3 acceptance tests pass; M3d merged; issue #15 closes.

## Risk Assessment

Overall residual risk: R3 High. M2 dominates: generated lifecycle behavior
(systemd units, firewall rules, auth environment) and a new archetype class in
the composition path that every machine composed from it inherits.

It is not R4: no PR touches key material or deployed state, every contract rule
is exercisable in checks and generated artifacts, and the worst credible failed
run after validation is a red example composition — recovery is a revert. That
residual R3 is why the acceptance tests lean on generated-artifact assertions
and mutation-style negatives rather than happy-path builds.

| Milestone | Risk | Reason | Human scrutiny |
|---|---|---|---|
| M1 package | R2 Medium | New derivation with no machine behavior; the uncertainty is build mechanics (MSRV forces the unstable rustPlatform; git-dep output hashes); straight revert. | Pin provenance: the rev and hash in the diff match the contract; the vendored lock and LICENSE copy; no floating ref anywhere in the expression. |
| M2 archetype + modules | R3 High | The module text is the security posture the first consumer inherits: fail-closed firewall rendering, unit preflight semantics, auth defaults. Blast radius is bounded — no machine composes the class until M3d — but a wrong default here ships silently in every service machine composed after it. | The rendered unit contract (ExecStartPre, non-optional EnvironmentFile, no ConditionPathExists), the firewall render and its assertions, and the drvPath parity output for existing machines. |
| M3 example + checks | R2 Medium | Synthetic data plus checks; the lockstep window is the main hazard, mitigated by the single-commit advance and pre-merge override validation; per-repo straight reverts. | The M3d lock diff moves exactly three inputs; the artifact-check output; the negative cases demonstrably failing. |

## Interface Contracts

### 1. Upstream facts the module must own

All observed in the upstream source at the pinned rev
`2ce2d71cc38a9657eaf344c10e07f155b8a18615` (github.com/block/buzz):

- **Signing key and auth are coupled.** The relay reads
  `BUZZ_RELAY_PRIVATE_KEY`. Unset with `BUZZ_REQUIRE_AUTH_TOKEN` false, it
  starts on a hardcoded deterministic dev keypair with only a log warning, and
  REST requests bypass token auth. Unset with the flag true, the process
  aborts at startup. Both dev-mode hazards — impersonatable relay identity and
  open REST — hang off the same flag.
- **S3 is mandatory at boot.** The relay runs a git object-store conformance
  probe against the configured S3/MinIO backend during startup and exits when
  it fails. MinIO is a hard boot dependency, not a lazily connected service.
- **Search is Postgres FTS.** A generated tsvector column with a GIN index
  inside the events table; no separate search service exists at this rev. The
  checked-in `.env.example` and compose files still reference a retired
  external search backend — they are not authoritative for the relay.
- **Three listeners.** The app router (WebSocket + REST + web) binds
  `BUZZ_BIND_ADDR` (upstream example `0.0.0.0:3000`); a health-only listener
  binds `0.0.0.0` at `BUZZ_HEALTH_PORT` (default 8080); Prometheus metrics
  bind `0.0.0.0` at `BUZZ_METRICS_PORT` (default 9102). Health and metrics
  ports are configurable but their bind addresses are not, so off-box exposure
  is entirely the firewall's job.
- **Migrations are in-process.** Schema migrations run at relay startup when
  `BUZZ_AUTO_MIGRATE` is truthy (default off). The module owns setting it so
  no manual migration step exists.
- **Backing-store env**: `DATABASE_URL`, `REDIS_URL`, `BUZZ_S3_ENDPOINT`,
  `BUZZ_S3_BUCKET` (upstream default `buzz-media`).
- **Packaging**: a plain cargo workspace; the relay binary is `buzz-relay`
  (crate `buzz-relay`), the CLI binary is `buzz` (crate `buzz-cli`);
  `rust-toolchain.toml` pins rustc 1.95.0; `Cargo.lock` carries two
  git-sourced dependencies; the repo is Apache-2.0 with no NOTICE file at this
  rev.

### 2. Composition contract

- `profileArchetypes = [ "dev" "privacy" "hypervisor" "service" ]`;
  `builders.service = mkServiceVm`.
- `mkServiceVm { name, identity, definition }` mirrors the privacy builder:
  platform from the machine facts, the shared VM baseline (qemu guest, agenix,
  host-key identity, home-manager), then `modules/service.nix` and
  `modules/buzz-stack.nix`, then `definition.nixosModules`;
  `definition.homeModules` are wired for `identity.username` on the shared
  stateVersion baseline. `profileData.<machine>` merges into builder args
  exactly as for every archetype.
- **No forge write capability by default**: `mkServiceVm` imports none of the
  dev builder's credential modules (`agent-forgejo-token.nix`, `netrc.nix`,
  `github-credentials.nix`) and threads no token files. A service machine that
  wants forge access must compose it deliberately in its own profile.
- Baseline inbound surface is SSH only: the shared guest baseline enables
  openssh and the NixOS firewall stays enabled. Everything else is opt-in
  through the allowlist below.
- Service machines flow into `vmFacts` with no vm-facts code change (its
  filter excludes only hypervisors), which is what makes provisioning-facts
  consumers work — and why M3a must supply host-key entries for the example
  machine.

### 3. Data-input contracts (secrets and inventory)

- `secrets.lib.serviceIdentities`: attrset `machine-name -> { username }`, the
  exact shape of `privacyIdentities`. The framework merges it into the
  identities set (`dev // privacy // service // { nexus }`) and extends the
  existing disjointness count assertion to cover it. When the secrets input
  does not export the attribute, the framework treats it as `{}`. That default
  cannot fail silently: a service machine without a matching identity — or the
  reverse — still dies at the existing `machines keys must exactly match
  identity keys` assertion, which is the loud guard that makes the default
  safe and the source of the M3 lockstep constraint.
- `secrets.lib.vmUsernames` must cover service machines (consumed by
  `vmFacts`).
- Inventory: `machines.service-1 = { platform = <the template's supported
  platform>; type = "service"; memory_mb = 2048; vcpus = 2; disk_gb = 20;
  ip = "192.0.2.12"; mac = "52:54:00:00:00:12"; forge_key = null; repos = [];
  self_rebuild = false; }`. It appears in `vmSpecsJson` — the filter excludes
  only hypervisors — so `scripts/vm-specs.json` regenerates in the same PR.

### 4. Service firewall allowlist

- Option: `allod.service.firewall.tcpAllow`, declared by
  `modules/service.nix`. Type `listOf (submodule { port : types.port;
  sources : nonEmptyListOf types.str; })`, default `[]`.
- Semantics: each entry renders one iptables accept rule per source into
  `networking.firewall.extraCommands` (`iptables -A nixos-fw -p tcp
  -s <source> --dport <port> -j nixos-fw-accept`) — the fleet's iptables
  backend. Ports in `tcpAllow` are never added to
  `networking.firewall.allowedTCPPorts`; that list is unscoped and would
  defeat the source restriction.
- Assertions the module carries, each with a negative check:
  `networking.firewall.enable` must be true when any entry exists (rules in
  `extraCommands` are vacuous behind a disabled firewall); a `tcpAllow` port
  must not also appear in `allowedTCPPorts`; `sources` is non-empty by type,
  so an empty source list is an eval error, not an open port.
- **How profile data reaches the firewall**: a machine's profile definition
  (`profileDefinitions.service.<name>.nixosModules` in the composed profiles
  input) is imported into the machine's module list by `mkServiceVm`, exactly
  as the dev and privacy builders import their definitions. Those modules set
  `allod.service.firewall.tcpAllow` — usually via the buzz options below — and
  the NixOS module system merges the values the firewall module renders.
  Per-machine allowlists are therefore ordinary composed profile data: no new
  channel, no builder argument.

### 5. Buzz stack module (`modules/buzz-stack.nix`)

Options, all under `services.buzz`:

| Option | Type, default | Meaning |
|---|---|---|
| `enable` | bool, `false` | The whole stack: Postgres, Redis, MinIO, relay unit. |
| `port` | port, `3000` | App-router listener (WS + REST + web); renders `BUZZ_BIND_ADDR`. |
| `allowedSourceIPs` | listOf str, `[]` | Sources allowed to reach `port`; feeds one `tcpAllow` entry when non-empty. Empty means closed. |
| `healthPort` | port, `8080` | Renders `BUZZ_HEALTH_PORT`. |
| `healthAllowedSourceIPs` | listOf str, `[]` | Same pattern for the health listener; default closed. |
| `metricsPort` | port, `9102` | Renders `BUZZ_METRICS_PORT`. |
| `metricsAllowedSourceIPs` | listOf str, `[]` | Same pattern for the metrics listener; default closed. |
| `environmentFile` | nullOr str, `null` | REQUIRED when `enable` (module assertion): absolute runtime path of an environment file defining `BUZZ_RELAY_PRIVATE_KEY`. A string, never a Nix path literal; an assertion rejects any value under the store prefix, because a store-copied key is world-readable. |
| `requireAuthToken` | bool, `true` | Renders `BUZZ_REQUIRE_AUTH_TOKEN` into the unit environment, module-owned — not read from `environmentFile`. |

Unit contract, `buzz-relay.service`:

- `EnvironmentFile=` references `environmentFile` in the non-optional form (no
  leading `-`). An `ExecStartPre` preflight exits non-zero unless the file
  exists, is non-empty, and defines `BUZZ_RELAY_PRIVATE_KEY`. On a missing or
  empty key the unit must land in systemd `failed` state — restart policy
  `on-failure` with the default start-limit, never an unbounded silent retry —
  and the unit must not use `ConditionPathExists=` on the key path: a failed
  condition skips the unit silently, which is the opposite of fail-loud.
- The key requirement is unconditional. Setting `requireAuthToken = false`
  relaxes neither the option, the assertion, nor the preflight — the upstream
  dev-keypair fallback must be unreachable from any built machine.
- NixOS activation completes with the key absent: the module adds no
  activation snippet for the key at all (delivery is downstream), honoring the
  public provisioning gotcha that activation snippets must never exit the
  shared activation script.
- Ordering: `buzz-relay.service` is wanted by `multi-user.target` and ordered
  after `postgresql.service`, `redis-buzz.service`, and `minio.service` with
  hard requirements — the S3 conformance probe makes MinIO a boot dependency.
  The module sets `BUZZ_AUTO_MIGRATE` truthy so schema setup is unit-owned.
- Backing services are native NixOS modules (`services.postgresql`,
  `services.redis.servers.buzz`, `services.minio`) — no containers — bound to
  loopback or UNIX sockets only. No credential material enters the Nix store:
  Postgres access prefers local socket peer auth for the relay's DB role (no
  password; if implementation forces loopback TCP, its password is
  machine-local runtime state), Redis is loopback-only, and the MinIO root
  credentials are machine-local runtime state generated on first start (0600,
  service-owned) and consumed by the relay through a generated runtime
  environment file — never option defaults, never store paths.

### 6. Package

- `packages.<system>.buzz` — one derivation building the workspace's
  `buzz-relay` and `buzz-cli` crates at the pinned rev; Apache-2.0.
  `packages.<system>.buzz-cli` — `bin/buzz` only, derived from the same build.
  Both exposed for each system in `inventory.lib.supportedPlatforms`.
- Built with `rustPlatform` from the `nixpkgs-unstable` input (MSRV fact
  above). The vendored `Cargo.lock` and upstream `LICENSE` copy live beside
  the package expression; the two git-sourced dependencies carry pinned
  `outputHashes`.
- The stack module consumes `bin/buzz-relay` from `packages.<system>.buzz`.

### 7. CLI channels

- **Dev leg**: `modules/dev-home-shared.nix` adds `buzz-cli` to
  `home.packages`, so the pinned `buzz` binary is on every dev VM PATH with no
  per-profile wiring — the same embedding pattern as the tools already built
  into that module.
- **Hypervisor leg**: `mkHypervisor` adds `buzz-cli` (the built package for
  the machine platform) to `home-manager.extraSpecialArgs` beside its existing
  entries, so hypervisor home modules can put the CLI on the host operator's
  PATH. This leg is contract, not convenience: a dev-only channel strands any
  deployment whose operator console is the hypervisor.

### 8. Exported names — the complete new-contract list

Everything a consumer may reference; a rename after landing is a breaking
interface change:

- `profileArchetypes` containing `"service"`; `builders.service`;
  `mkServiceVm { name, identity, definition }`.
- `packages.<system>.buzz`, `packages.<system>.buzz-cli`.
- Option paths: `allod.service.firewall.tcpAllow`; `services.buzz.{enable,
  port, allowedSourceIPs, healthPort, healthAllowedSourceIPs, metricsPort,
  metricsAllowedSourceIPs, environmentFile, requireAuthToken}`.
- Unit names: `buzz-relay.service`, `postgresql.service`,
  `redis-buzz.service`, `minio.service`.
- Secrets-input exports: `lib.serviceIdentities`; `lib.vmUsernames` covering
  service machines.
- `home-manager.extraSpecialArgs.buzz-cli` in `mkHypervisor`.
- Check names: `service-stack-contracts`, `service-machine-artifacts` under
  `checks.<system>` for each supported platform.

## Agent Gates

- Public repo pushes are relay-only: the agent prepares commits and PR bodies;
  a maintainer applies and pushes them. Blocks every PR in the arc.
- PR merges are human, throughout. The M3 lockstep constraint is a merge-time
  discipline the human enforces: no routine single-input lock advance of
  `inventory`, `secrets`, or `profiles` in `allod/archetypes` between the
  first M3 template merge and M3d.
- No real key material exists anywhere in this plan. The example machine's
  `environmentFile` is a documented runtime path a deploy fork points at its
  own delivery mechanism; nothing blocks on it.

## Acceptance Tests

Run in the repo each milestone changes. `system="$(nix eval --raw --impure
--expr builtins.currentSystem)"` avoids literal system strings. Whole-flake
`nix flake check` on the framework evaluates every machine configuration in
one process; fall back to per-check builds if memory-tight (public memory
`nix.md`).

```sh
# ── M1: package ─────────────────────────────────────────────────────────
nix build .#packages."$system".buzz -o result-buzz
nix build .#packages."$system".buzz-cli -o result-cli
test -x result-buzz/bin/buzz-relay && test -x result-buzz/bin/buzz
test -x result-cli/bin/buzz && test ! -e result-cli/bin/buzz-relay
result-cli/bin/buzz --help >/dev/null
git grep -n '2ce2d71cc38a9657eaf344c10e07f155b8a18615' packages/buzz/  # pin in-tree
test -f packages/buzz/LICENSE && grep -q 'Apache License' packages/buzz/LICENSE

# ── M2: archetype + modules ─────────────────────────────────────────────
# drvPath parity: run this loop at the base rev and on the PR branch; every
# machine's drvPath must be byte-identical (no lock moves in this PR).
for m in $(nix eval .#nixosConfigurations --apply builtins.attrNames --json | jq -r '.[]'); do
  nix eval .#nixosConfigurations."$m".config.system.build.toplevel.drvPath
done
nix build .#checks."$system".service-stack-contracts
# That check counts only because its negatives demonstrably fail. Via tryEval
# over synthetic per-platform service evaluations it must assert that each of
# these FAILS evaluation — and that the well-formed evaluation succeeds:
#   (a) services.buzz.enable with environmentFile unset
#   (b) environmentFile given as a store path
#   (c) a tcpAllow entry with an empty sources list
#   (d) a tcpAllow port duplicated into networking.firewall.allowedTCPPorts
#   (e) tcpAllow entries present with networking.firewall.enable forced false
# Existing checks stay green:
for c in $(nix eval .#checks."$system" --apply builtins.attrNames --json | jq -r '.[]'); do
  nix build .#checks."$system"."$c"
done

# ── M3a: secrets template ───────────────────────────────────────────────
nix flake check       # credential-inventory covers the new host-key file
nix eval --json .#lib.serviceIdentities | jq -e '."service-1".username == "service"'
nix eval --json .#lib.vmUsernames | jq -e '."service-1" == "service"'
jq -e '."service-1".active | startswith("ssh-ed25519 ")' machine-host-keys.json
test -f secrets/vm-host-keys/service-1-ssh.age

# ── M3b: inventory template ─────────────────────────────────────────────
nix flake check       # vm-specs-json + repository-registry
nix eval --json .#machines.service-1 \
  | jq -e '.type=="service" and .ip=="192.0.2.12" and .forge_key==null'
jq -e '."service-1".self_rebuild == false' scripts/vm-specs.json

# ── M3c: profiles ───────────────────────────────────────────────────────
nix flake check
nix eval --json .#lib.profileDefinitions | jq -e '.service | keys == ["service-1"]'

# ── M3d: archetypes lock advance + artifact checks ──────────────────────
# Pre-merge: the same evals with --override-input against the branch revs
# (quote the URL; the no-lock-write warning is expected).
nix eval .#nixosConfigurations.service-1.config.system.build.toplevel.drvPath
nix eval --json .#vmFacts.service-1 | jq -e '.ip == "192.0.2.12"'

# Firewall: enabled, source-scoped, nothing unscoped.
nix eval --json .#nixosConfigurations.service-1.config.networking.firewall.enable \
  | jq -e '. == true'
nix eval --json .#nixosConfigurations.service-1.config.networking.firewall.allowedTCPPorts \
  | jq -e '([3000,8080,9102] - .) | length == 3'   # none opened unscoped
nix eval --raw .#nixosConfigurations.service-1.config.networking.firewall.extraCommands > /tmp/fw
grep -E -- '-s 192\.0\.2\.10\b.*--dport 3000|--dport 3000.*-s 192\.0\.2\.10\b' /tmp/fw
! grep -E -- '--dport (8080|9102)' /tmp/fw        # health/metrics closed in the example
! grep -E '192\.0\.2\.0/24' /tmp/fw               # never a whole-subnet rule

# Generated units.
nix build .#nixosConfigurations.service-1.config.system.build.toplevel -o result-svc
unit=result-svc/etc/systemd/system/buzz-relay.service
grep -E '^ExecStartPre=' "$unit"                  # the key preflight exists
grep -E '^EnvironmentFile=/' "$unit"              # non-optional form
! grep -E 'EnvironmentFile=-' "$unit"             # optional form is the dev-key trap
! grep -E 'ConditionPathExists=' "$unit"          # condition-skip is silent, reject it
# Auth default, rendered inline or via a referenced store-side env file:
grep -q 'BUZZ_REQUIRE_AUTH_TOKEN=true' "$unit" \
  || grep -q 'BUZZ_REQUIRE_AUTH_TOKEN=true' \
       $(grep -oE '/nix/store/\S+' "$unit" | sort -u)
ls result-svc/etc/systemd/system/ | grep -E '^postgresql'   # one grep per unit —
ls result-svc/etc/systemd/system/ | grep -E '^redis-buzz'   # an alternation passes when
ls result-svc/etc/systemd/system/ | grep -E '^minio'        # ANY one exists, hiding a miss

nix build .#checks."$system".service-machine-artifacts   # the checked-in form of the above
```

Negative path, stated precisely: with `services.buzz.enable = true` and
`environmentFile` unset, the machine **fails evaluation** (module assertion) —
it does not build a degraded unit. The runtime case — option set, file absent
or empty on the running machine — is the fail-loud unit above: `ExecStartPre`
exits non-zero, the unit lands in `failed`, and activation still completes.
`service-stack-contracts` pins the eval-time case; `service-machine-artifacts`
pins the unit text of the runtime case.

## Rollback Plan

Reverts run in reverse dependency order; each milestone is a straight revert
of its PRs.

- **M1**: revert; the `packages.*` outputs disappear. After M2 lands, revert
  M2 first — the stack module and the home embeddings consume the package.
- **M2**: revert restores the `mkServiceVm` stub. While M3d is unmerged no
  machine composes the class, so nothing else changes. After M3d, revert M3d
  first so the locks stop referencing service data before the class vanishes.
- **M3**: revert M3d first — it is the only commit that makes the framework
  consume the template entries; the example machine and its checks disappear
  and the locks return to pre-service revs. Then M3a/b/c revert independently
  (the inventory revert regenerates `scripts/vm-specs.json`; its check forces
  this). Reverting a template PR while M3d stands leaves the framework green
  at its current lock but red at its next lock advance — the lockstep rule in
  reverse — so never leave that state in place.
- **Full-arc abandonment**: revert M3d, M3c, M3b, M3a, M2, M1 in that order.
  The stub comment returns; `allod/deploy` needs nothing, since it re-exports
  whatever the framework composes at its own lock. While no downstream deploy
  fork composes a service machine, these reverts break no consumer anywhere;
  once the first fork adopts the class, the exported names in Interface
  Contracts 8 are frozen and their removal is a breaking change negotiated on
  issue #15.
