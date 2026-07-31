# Archetypes Consumed-File Carve-Out

## Tracking Issue

`allod/archetypes#21` — "Carve out consumed files so a dev machine stops carrying the whole secrets repo". Single PR; it carries `Closes allod/archetypes#21`.

## Goal

A dev machine's system closure holds only the individual files it consumes from the secrets input, so one guest compromise no longer yields a copy of every ciphertext in the deployment.

## Scope

In scope:

- `modules/agent-hooks.nix:14,16,18,20` — the four policy files selected from `gitPolicySource`, which defaults to the secrets input.
- `modules/github-credentials.nix:29` — `file = "${secrets}/${consumer.secret}"`.
- `flake.nix:494` — `githubTargetMatchesGeneratedAgeSecret`, which rejects a carved path today and must bind the generated secret to the exact carved source.
- A new check that fails if any in-scope consumer retains the input root, and that exercises the GitHub path with synthetic non-empty target data.

Out of scope:

- The recipient graph in `secrets.nix`. Which machine can decrypt which ciphertext is correct and untouched; this plan changes exposure, not access.
- The check-derivation references at `flake.nix:717` and `:723`. A check that inspects the repo is supposed to see the repo, and it never lands on a machine.
- `modules/agent-hooks.nix:11` and the `allod-tools` input generally. The tools input contains no ciphertext, so removing its source root is derivation-churn work rather than this issue's exposure boundary. In the runtime closure this plan measures, the tools root is held only by that one site — home-manager symlinks a `home.file` source whose executable bit already matches, and the symlink target names the root. `modules/dev-home-shared.nix:25` and the `builtins.readFile` consumers reach the root only through the build closure, which never lands on a machine — the same build/runtime distinction that already exempts the check references. Do the tools carve, if still useful, as a separate change with its own closure claim.
- `modules/agent-forgejo-token.nix:5` and `flake.nix:199`. Their values originate as `./secrets + "/<name>.age"`, which is a Nix path, not a string naming a child of the input root. Coercing that path for `age.secrets.<name>.file` already copies the individual file to `/nix/store/<hash>-<name>.age`; these sites do not retain the secrets source root.
- `mkPrivacyVm` and `mkHypervisor`. Neither imports the modules in scope.
- The identity template (`allod/secrets`) and every fork's data repos. No new export, no data-repo change. See Interface Contracts.

## Risk Assessment

Residual risk: R3 High

Why:

- Blast radius is every dev machine at once: each carve changes `age.secrets.<name>.file` or a `home.file.<name>.source`, so every dev machine's derivation moves. Privacy VMs and the hypervisor are unaffected — the modules in scope are imported only from `mkDevVm` (`flake.nix:159-218`).
- It is a secret-handling boundary change, which `architecture.md` ranks above everything else, and it alters activation-time behavior: agenix decrypts the GitHub credential from `age.secrets.<name>.file` and home-manager places four policy files. A happy-path evaluation can pass while activation fails, which is the lesson principle 13 says was already paid for once.
- A silent increase in exposure is credible even without touching recipients: a wrong `builtins.path` source can put an unintended ciphertext in the machine closure while the secrets root is absent. The closure test alone cannot catch that. Exact source equality for every generated consumer, including a same-basename wrong-content mutation, is therefore part of the security proof.
- What keeps the residual risk at R3 rather than R4 is that the implementation mutates no authoritative secret, recipient graph, key, or persistent data; the exact-source and generated-metadata checks are agent-runnable; the built output closure is mechanically inspectable; rollout starts on a disposable VM; and rollback is a previous generation or a pin/revert plus rebuild. A bad source, destination, owner, group or mode is caught before rollout, while a missed site leaves today's exposure and fails the output-closure check.
- The central acceptance test queries a **built toplevel output**, not its derivation. The secrets source directory either is or is not in the runtime closure that is copied to the machine. That is binary and mechanical rather than a judgement about the diff.

Human scrutiny, in order:

1. The built-output closure query. It proves the whole source repo no longer reaches a machine, but must be read together with the exact-source allowlist check.
2. The `credential-profiles` matcher and its same-basename mutation at `flake.nix:494` — the change must strengthen the binding to exact equality, not merely make a carved suffix pass.
3. Activation on a throwaway machine before any machine you work from takes it.

## Interface Contracts

**The carve construction.** Every file consumer in scope becomes:

```nix
builtins.path { path = <existing interpolated path>; name = <basename>; }
```

This yields a content-addressed store path holding the single file, with no reference back to the source directory. (References discovered from the file's own bytes are orthogonal; the contract is that the input root is absent.) It behaves the same whether the argument is a literal path or a string carrying the input's context, so no context-discarding helper is needed.

**Carve in the consuming modules, not via a new identity-template export.** The issue leaves this open. Decided: consuming modules. It is confined to `archetypes`, needs no new export in the identity template and no matching change in any fork's data repo.

**The token files are already individual store paths.** `secrets/flake.nix:49-56` builds `forgeTokenFile` and `agentTokenFile` as `./secrets + "/<name>.age"`. That expression evaluates to a Nix path. On Nix 2.31.5, evaluating the expression against the checked-out secrets tree confirms the distinction:

- `builtins.typeOf (./secrets + "/agent-pr-token.age")` is `"path"`;
- interpolating that result produces `/nix/store/<hash>-agent-pr-token.age`;
- interpolating `./secrets` first and then appending the filename produces `/nix/store/<hash>-secrets/agent-pr-token.age`.

The first form is what the identity template exports, so `modules/agent-forgejo-token.nix:5` and `flake.nix:199` already consume a single-file store object. Do not wrap them in a second `builtins.path`. Note that `toString` performs no copy at all: on these path values it names the child inside the source root (`/nix/store/<hash>-source/secrets/<name>.age`), so any check that projects `age.secrets.<name>.file` must coerce with interpolation — the coercion agenix itself performs when it embeds the file in the activation script — never `toString`.

**`credential-profiles` must bind the GitHub target to the exact carved file.** This is the one place where following the issue literally breaks a check once non-empty GitHub target data is composed.

| matcher | location | accepts a carved path? |
|---|---|---|
| `targetMatchesGeneratedAgeSecret` (Forgejo) | `flake.nix:456-457` | yes — `hasSuffix "/${target.secretPath}"` **or** `hasSuffix "-${secretFileName}"` |
| `githubTargetMatchesGeneratedAgeSecret` (GitHub) | `flake.nix:494,499` | no — `hasSuffix "/${consumer.secret}"` only |

A carved file is `/nix/store/<hash>-<basename>.age`, which has no `/secrets/<name>.age` suffix. Do **not** copy the Forgejo matcher's basename fallback: a suffix proves only the label, so an unrelated carved file with the same basename also passes. The GitHub matcher already selects `config.age.secrets.${target.credential}` and checks destination, owner, group and mode. Complete that binding by independently constructing the expected source:

```nix
expectedFile = builtins.path {
  path = "${secrets}/${consumer.secret}";
  name = baseNameOf consumer.secret;
};
```

Require `toString ageSecret.file == toString expectedFile`; remove the old relative-path suffix condition rather than retaining a compatibility branch. There is one GitHub construction site and backwards compatibility is not required.

**The name argument is part of the exact contract.** Every carve uses the source basename. Two same-named files with different content have different store hashes; identical content may share a store object. Either way, exact expected-path equality needs no uniquification.

**The regression check exercises generated values, not source spelling.** For the four policy files, assert that the generated `home.file` sources equal independently constructed single-file paths for the declared policy files. Lift the GitHub file/metadata comparison into a small local predicate that accepts a target, consumer, generated secret and secrets source; the real matcher resolves those inputs from `machineConfigurations`, and the fixture calls the same predicate directly. Evaluate `modules/github-credentials.nix` with a synthetic non-empty credential/target input, assert the exact generated file/path/owner/group/mode, and feed that result through the predicate. The module takes no `config` and uses no `mkIf`, so the fixture is direct function application — `(import ./modules/github-credentials.nix { secrets = <synthetic>; machineName = <fixture>; }) { inherit lib; }` — with no builder refactor. A same-basename, different-content carved file must be a failing mutation. Also cover an empty target list and a target with no unique agenix consumer, so the optional and fail-loud paths do not go vacuous. This catches concatenation or variable-renaming regressions that a grep for `"${secrets}/..."` would miss.

The fixture's secret bytes must exist at evaluation time: `builtins.path` reads its source during eval, so a derivation-built fixture is import-from-derivation and fails. Build the synthetic input as the real `secrets` input with a substituted `lib` — interpolation still resolves through `outPath` — and point the consumer at an existing template ciphertext such as `secrets/agent-pr-token.age`. The same-basename wrong-content mutation then needs no new file either: `builtins.path { path = "${secrets}/secrets/forgejo-https-token-allod-dev.age"; name = "agent-pr-token.age"; }` carves different bytes under the colliding name.

## Agent Gates

- **Merging, advancing the private deploy pin, and rebuilding or replacing a machine are human-only.** The agent cannot merge the framework PR, update the private deployment for rollout, run `rebuild-vm-from-host`, or run `provision-vm-from-host`. This blocks acceptance test 8. Tests 1–7 are agent-runnable against an input override and do not require a lock advance.
- **The throwaway machine goes first.** This change runs at activation on every dev VM, including the machine the agent is working from, so proving it by rebuilding the working machine risks the seat needed to fix it. The human rebuilds a disposable machine first and the working machine only after.
- No secret is read, written, re-encrypted, or rekeyed by this change, so no agenix or key ceremony is involved.

## Acceptance Tests

```sh
# Run from the candidate archetypes checkout. BASE_ARCHETYPES is a clean
# checkout of the pre-change revision; DEPLOY is the composition root.
set -euo pipefail
ARCHETYPES=$PWD
WORKSPACE=${WORKSPACE:-"$HOME/work"}
BASE_ARCHETYPES=$WORKSPACE/allod/archetypes
DEPLOY=$WORKSPACE/allod/deploy
M=allod-dev
SYSTEM=$(nix eval --raw "$DEPLOY#nixosConfigurations.$M.pkgs.stdenv.hostPlatform.system")
SECRETS_STORE=$(nix eval --raw --impure --expr \
  "(builtins.getFlake \"path:$DEPLOY\").inputs.secrets.outPath")

# 1. BEFORE: query the realised system output, not its .drv build closure.
base_out=$(nix build --no-link --print-out-paths \
  "$DEPLOY#nixosConfigurations.$M.config.system.build.toplevel" \
  --override-input archetypes "path:$BASE_ARCHETYPES")
nix-store -q --requisites "$base_out" \
  | grep -F -x -- "$SECRETS_STORE" >/dev/null \
  || { echo "ERROR: baseline closure lacks $SECRETS_STORE — the baseline is not pre-change or the premise is wrong; stop here" >&2; exit 1; }

# 2. AFTER: the same runtime closure no longer contains the secrets source root.
candidate_out=$(nix build --no-link --print-out-paths \
  "$DEPLOY#nixosConfigurations.$M.config.system.build.toplevel" \
  --override-input archetypes "path:$ARCHETYPES")
if nix-store -q --requisites "$candidate_out" \
  | grep -F -x -- "$SECRETS_STORE"; then
  echo "ERROR: candidate system closure still contains $SECRETS_STORE" >&2
  exit 1
fi

# Repeat test 2 for every composed dev configuration. The public template
# composes exactly one (allod-dev, covered above); the private composition runs
# this same block once per dev machine, including every one with a GitHub target.

# 3. The generated-value check proves the four policy sources and the synthetic
#    GitHub file/path/owner/group/mode projection exactly, including empty,
#    invalid-consumer, and same-basename wrong-content mutations.
nix build "$ARCHETYPES#checks.$SYSTEM.consumed-file-carve-out" --no-link

# 4. Existing credential and generated agenix lifecycle checks still pass. The
#    public secrets template declares no GitHub targets, so credential-profiles
#    alone is not evidence for the new matcher; test 3 supplies the non-empty fixture.
nix build "$ARCHETYPES#checks.$SYSTEM.credential-profiles" --no-link
nix build "$ARCHETYPES#checks.$SYSTEM.dev-forge-opt-out" --no-link

# 5. Prove the new check rejects a real consumer regression: un-carve one policy
#    source in place, require the check build to fail naming that target, then
#    restore. Restore from the scratch copy, never `git checkout --`, which
#    would clobber uncommitted implementation work. The sed assumes each carved
#    consumer stays on one line; if the sabotage instead produces an eval error,
#    the wrong-reason branch below catches it.
orig=$(mktemp)
cp modules/agent-hooks.nix "$orig"
sed -i 's|home\.file\.".config/git/protected-branches"\.source = .*|home.file.".config/git/protected-branches".source = "${gitPolicySource}/git/protected-branches";|' \
  modules/agent-hooks.nix
if out=$(nix build "$ARCHETYPES#checks.$SYSTEM.consumed-file-carve-out" --no-link 2>&1); then
  mv "$orig" modules/agent-hooks.nix
  echo "ERROR: consumed-file check passed with an uncarved policy source" >&2
  exit 1
fi
mv "$orig" modules/agent-hooks.nix
case "$out" in
  *protected-branches*) ;;
  *)
    echo "ERROR: consumed-file check failed, but not on the sabotaged target:" >&2
    printf '%s\n' "$out" >&2
    exit 1
    ;;
esac

# 6. Inspect each composed dev machine's age secret paths, coerced the way
#    agenix embeds them (string interpolation). A correct consumer — a token
#    path value or a carved file — materialises as a regular store-root file
#    (/nix/store/<hash>-<basename>, no child path); an uncarved
#    "${secrets}/..." string keeps its in-root spelling and fails. toString
#    would misreport the token files' pre-coercion location
#    (/nix/store/<hash>-source/secrets/<name>.age) and fail every
#    forge-access machine spuriously. Compare the exact
#    destination/owner/group/mode projection through test 3 or the private
#    composition's corresponding credential contract check.
secret_files=$(nix eval --json \
  "$DEPLOY#nixosConfigurations.$M.config.age.secrets" \
  --override-input archetypes "path:$ARCHETYPES" \
  --apply 's: builtins.mapAttrs (_: v: "${v.file}") s' \
  | jq -r '.[]')
while IFS= read -r file; do
  case "$file" in
    /nix/store/*/*|"")
      echo "ERROR: age secret is not a store-root file: $file" >&2
      exit 1
      ;;
    /nix/store/*) ;;
    *)
      echo "ERROR: age secret is outside the store: $file" >&2
      exit 1
      ;;
  esac
  if [ ! -f "$file" ]; then
    echo "ERROR: age secret is not a regular file: $file" >&2
    exit 1
  fi
done <<< "$secret_files"

# 7. Every configuration still evaluates, one process per configuration. Never
#    run nix flake check on the composition root: it peaks near 7 GiB on an 8 GiB VM.
for cfg in $(nix eval --json "$DEPLOY#nixosConfigurations" \
    --apply builtins.attrNames | jq -r '.[]'); do
  nix eval --raw \
    "$DEPLOY#nixosConfigurations.$cfg.config.system.build.toplevel.drvPath" \
    --override-input archetypes "path:$ARCHETYPES"
  echo
done
```

8. **Human generated-lifecycle gate:** after merge, the human advances the private deploy pin and rebuilds a disposable dev VM that has its real GitHub target. Confirm agenix activation decrypts the declared credential to the unchanged destination and home-manager installs all four policy files with readable targets. Reboot once, then rebuild the working VM only if both activation and reboot are clean.

Test 3 is what proves the carved GitHub source is still bound to the declared target. The public `credential-profiles` target list is empty today, and the deploy template re-exports only `composed-layer`, so `nix build "$DEPLOY#checks.$SYSTEM.credential-profiles"` is not a substitute. Test 1 is not ceremony: if it comes back zero, the premise for the four policy consumers is wrong and implementation should stop rather than proceed to a vacuously passing test 2.

## Rollback Plan

No state migrates and no secret is rewritten. Revert the single framework PR, then return the private deploy pin to the pre-change archetypes revision (or advance it to the revert) before rebuilding any affected VM. A machine already switched to the candidate can boot its previous NixOS generation immediately; rebuilding from the restored pin returns the declared behavior.

Partial states:

- **Reverting after merge but before a deploy-pin advance** costs nothing. No machine references the change.
- **Reverting after the deploy pin advances but before a rebuild** requires restoring the pin, but changes no machine state.
- **Reverting after a machine has rebuilt** is a rebuild back to the previous generation, or a NixOS generation rollback at the console if activation is what broke. Dev VMs are disposable; a machine that will not activate is replaced from a clone rather than repaired.
- **A failed activation can leave derivative files partially updated.** Do not repair them by hand. Boot the previous generation or replace the disposable VM, restore the deploy pin, and rebuild from declared sources.
