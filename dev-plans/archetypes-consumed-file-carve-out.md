# Archetypes Consumed-File Carve-Out

## Tracking Issue

`allod/archetypes#21` — "Carve out consumed files so a dev machine stops carrying the whole secrets repo". Single PR; it carries `Closes allod/archetypes#21`.

## Goal

A dev machine's system closure holds only the individual files it consumes from the secrets and tools inputs, so one guest compromise no longer yields a copy of every ciphertext in the deployment.

## Scope

In scope:

- `modules/agent-hooks.nix` — five interpolations of an input root: `:11` (`allod-tools`) and `:14,16,18,20` (`gitPolicySource`).
- `modules/github-credentials.nix:29` — `file = "${secrets}/${consumer.secret}"`.
- `modules/agent-forgejo-token.nix:5` and `flake.nix:199` — the two agenix secrets whose `file` arrives from the identity template as `tokenFile` / `httpsTokenFile`. These are not named in the issue; see Interface Contracts for why they are load-bearing.
- `flake.nix:494` — `githubTargetMatchesGeneratedAgeSecret`, which rejects a carved path today and must accept one.
- A new check that fails if any of these sites reintroduces a bare interpolation of an input root.

Out of scope:

- The recipient graph in `secrets.nix`. Which machine can decrypt which ciphertext is correct and untouched; this plan changes exposure, not access.
- The check-derivation references at `flake.nix:717` and `:723`. A check that inspects the repo is supposed to see the repo, and it never lands on a machine.
- `mkPrivacyVm` and `mkHypervisor`. Neither imports the modules in scope. The hypervisor identity interpolates its own source the same way; that is a separate change in a different function, and the host is already a recipient of nearly everything it carries.
- The identity template (`allod/secrets`) and every fork's data repos. No new export, no data-repo change. See Interface Contracts.

## Risk Assessment

Residual risk: R3 High

Why:

- Blast radius is every dev machine at once: each carve changes `age.secrets.<name>.file` or a `home.file.<name>.source`, so every dev machine's derivation moves. Privacy VMs and the hypervisor are unaffected — the modules in scope are imported only from `mkDevVm` (`flake.nix:159-218`).
- It is a secret-handling boundary change, which `architecture.md` ranks above everything else, and it alters activation-time behavior: agenix decrypts from `age.secrets.<name>.file` and home-manager places four policy files. A happy-path evaluation can pass while activation fails, which is the lesson principle 13 says was already paid for once.
- What keeps it off R4: the failure modes are loud or inert, not silently worse. A wrong carve makes agenix fail at activation, and a missed site leaves exactly today's exposure. A mistake that *increases* exposure is hard to construct, because decryptability is governed by the recipient graph, which this does not touch. Dev VMs are disposable and rebuild from a clone, so rollback is a straight revert plus a rebuild.
- The central acceptance test is unusually decisive: the secrets source directory either is or is not in the closure. That is binary and mechanical rather than a judgement about the diff.

Human scrutiny, in order:

1. The closure query output. It is the only evidence that the security property actually holds; everything else in this plan is a means to it.
2. The `credential-profiles` matcher change at `flake.nix:494` — it loosens a check that guards credential binding, and loosening the wrong way would let a real mismatch pass.
3. Activation on a throwaway machine before any machine you work from takes it.

## Interface Contracts

**The carve construction.** Every site in scope becomes:

```nix
builtins.path { path = <existing interpolated path>; name = <basename>; }
```

This yields a content-addressed store path holding the single file with an empty reference set — no path back to the source directory. It behaves the same whether the argument is a literal path or a string carrying the input's context, so no context-discarding helper is needed.

**Carve in the consuming modules, not via a new identity-template export.** The issue leaves this open. Decided: consuming modules. It is confined to `archetypes`, needs no new export in the identity template and no matching change in any fork's data repo, and — decisively — it works regardless of how the template constructs the path. That last point is what brings `tokenFile` and `httpsTokenFile` into reach without touching `allod/secrets` at all.

**The token files are in scope even though the issue omits them.** `secrets/flake.nix:49-56` builds `forgeTokenFile` and `agentTokenFile` as `./secrets + "/<name>.age"`. Inside a flake, a relative path resolves within the flake's own store directory, so those values are `<secrets-store-path>/secrets/<name>.age` — the same whole-directory dependency by a different spelling. They reach a machine through `mkDevVm`'s `tokenFile` / `httpsTokenFile` arguments and land at `modules/agent-forgejo-token.nix:5` and `flake.nix:199`. Carving only the two modules the issue names would leave the secrets source directory in the closure through these, and the acceptance test below would fail. Implementation must confirm this by querying the closure before changing anything, so the starting position is measured rather than assumed.

**`credential-profiles` must accept a carved path — one matcher does, the other does not.** This is the one place where following the issue literally breaks the build.

| matcher | location | accepts a carved path? |
|---|---|---|
| `targetMatchesGeneratedAgeSecret` (Forgejo) | `flake.nix:456-457` | yes — `hasSuffix "/${target.secretPath}"` **or** `hasSuffix "-${secretFileName}"` |
| `githubTargetMatchesGeneratedAgeSecret` (GitHub) | `flake.nix:494,499` | no — `hasSuffix "/${consumer.secret}"` only |

A carved file is `/nix/store/<hash>-<basename>.age`, which has no `/secrets/<name>.age` suffix. The GitHub matcher guards exactly the module the issue names, so it must gain the same `-<basename>` alternative the Forgejo matcher already has. Keep both alternatives rather than replacing the path form: the check must still pass for any site not yet carved.

**The name argument is load-bearing.** `name` must be the file's basename, because the `-<basename>` suffix match is what binds a carved secret back to its declared target. A carve with a different name silently defeats the matcher.

**Naming collisions are acceptable.** Two different files carved to the same basename produce two distinct store paths; `builtins.path` keys on content, and the name is a label. No uniquification is needed.

## Agent Gates

- **Rebuilding or replacing a machine is host-side and human-only.** The agent cannot run `rebuild-vm-from-host` or `provision-vm-from-host`. This blocks the activation evidence in acceptance test 5, which is the only test that exercises the generated activation path rather than the evaluated closure. Everything else in this plan is agent-runnable.
- **The throwaway machine goes first.** This change runs at activation on every dev VM, including the machine the agent is working from, so proving it by rebuilding the working machine risks the seat needed to fix it. The human rebuilds a disposable machine first and the working machine only after.
- No secret is read, written, re-encrypted, or rekeyed by this change, so no agenix or key ceremony is involved.

## Acceptance Tests

```sh
# Run from an archetypes checkout with a deploy flake that composes it.
# M = a dev machine name; SECRETS_STORE = the secrets input's store path.

# 1. BEFORE: record the starting position, so "it disappeared" is measured, not assumed.
drv=$(nix eval --raw ".#nixosConfigurations.$M.config.system.build.toplevel.drvPath")
nix-store -q --requisites "$drv" | grep -c -- "$SECRETS_STORE"   # expect: non-zero

# 2. AFTER: the secrets source directory is gone from the closure entirely.
drv=$(nix eval --raw ".#nixosConfigurations.$M.config.system.build.toplevel.drvPath")
nix-store -q --requisites "$drv" | grep -- "$SECRETS_STORE"      # expect: no output, exit 1
# Same for the tools input's store path.

# 3. Every carved site resolves to a single-file store path, not a directory.
nix eval --json ".#nixosConfigurations.$M.config.age.secrets" \
  --apply 's: builtins.mapAttrs (_: v: toString v.file) s'
# expect: every value matches ^/nix/store/[a-z0-9]{32}-[^/]+$ with no trailing directory component

# 4. Content, destination, owner, group and mode are unchanged for all five policy
#    files and every credential. Compare the evaluated attrset against the pre-change
#    run field by field; only the `file`/`source` store path may differ.

# 5. credential-profiles still binds every declared target.
nix build ".#checks.<system>.credential-profiles" --no-link

# 6. The new anti-regression check fails on sabotage (principle 11).
#    Reintroduce a bare `"${secrets}/..."` interpolation at one site and confirm the
#    check fails and names the site. A check that cannot be shown to fail does not count.

# 7. Whole-flake evaluation still succeeds, per-configuration to stay inside memory.
#    Never `nix flake check` the composition root: it peaks near 7 GiB and is OOM-killed
#    on an 8 GiB VM (nix.md).
```

Test 5 is the one that proves the carve did not break credential binding, and it is the test that fails today if `flake.nix:494` is left alone. Test 1 is not ceremony: if it comes back zero, the premise of this plan is wrong and implementation should stop and say so rather than proceed to a vacuously passing test 2.

## Rollback Plan

Straight revert of the single PR. No state migrates, no secret is rewritten, and no lock advances, so a revert restores the previous derivations exactly.

Partial states:

- **Reverting after merge but before any rebuild** costs nothing. Merging changes no machine until the private deploy lock advances and a rebuild runs.
- **Reverting after a machine has rebuilt** is a rebuild back to the previous generation, or a NixOS generation rollback at the console if activation is what broke. Dev VMs are disposable; a machine that will not activate is replaced from a clone rather than repaired.
- **A half-applied carve cannot persist.** All sites land in one commit, and the new check fails the build if any site regresses, so there is no state where some paths are carved and others silently are not.
