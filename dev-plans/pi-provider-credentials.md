# Managed Pi Provider Credentials

This adds a Nexus-only enrollment command and a declarative dev-VM integration for custom Pi providers, while keeping bearer tokens encrypted at rest and out of shell arguments, environment variables, command output, persistent VM storage, and the Nix store. Implementation starts only after the owner approves this plan.

## Tracking Issue

allod/strategy#34. Every public implementation PR uses `Refs allod/strategy#34`; no PR closes it. The owner closes the issue only after the private live canary succeeds.

## Goal

The owner can register an HTTPS OpenAI-compatible endpoint on Nexus, paste its bearer token once, and rebuild one libvirt dev VM whose Pi process reads the token directly from an Age-backed runtime credential.

## Scope

In scope:

- `allod/secrets`: an empty public Pi-provider registry, its exported/validated interface, derived credential inventory entries, derived Age recipients, and synthetic checks.
- `allod/archetypes`: the dev-builder seam, libvirt Age-secret projection, a generated Pi provider extension, absence behavior, and generated-artifact/request witnesses.
- `allod/nexus`: a `pi-provider` command with `add`, `rotate`, and `--dry-run`, plus fixture tests and package/check wiring.
- `allod/deploy`: pin the compatible public revisions and run the composition canary.

Out of scope:

- Live URLs, model names, token values, or machine-specific provider entries in public history.
- OAuth, custom headers, non-bearer auth, non-OpenAI adapters, HTTP endpoints, automatic model discovery, token creation/revocation, or provider deletion.
- Privacy/service VM targets or microVM delivery. A non-libvirt target fails before secret input; microVM support waits for allod/archetypes#29 and allod/archetypes#38.
- Automatically deleting an old Pi `auth.json` entry. The owner retires that fallback only after the live managed path works.

Ordered public slices:

1. `secrets` defines the data contract and recipient derivation.
2. `archetypes` consumes that contract and proves runtime delivery with synthetic data.
3. `nexus` writes the contract safely and prints the human-only deployment sequence.
4. `deploy` pins the compatible revisions. These slices can land independently in that order.

## Risk Assessment

Residual risk: R3 High

The change crosses repository contracts and handles authentication, Age recipients, generated Home Manager behavior, and NixOS activation. Validation can prove the public mechanism with synthetic credentials, including a real local Pi request, but only the owner on Nexus can prove the private recipient graph, VM rebuild, and remote endpoint behavior. The old Pi credential remains available until the new live path passes, so a failed rollout is recoverable without rotating or losing the provider token.

Human scrutiny should focus first on token transport, the generated Pi auth resolver, recipient completeness, and the order of private pin update, rebuild, verification, and old-auth retirement.

| PR or milestone | Risk | Reason | Human scrutiny |
|---|---|---|---|
| `secrets` interface | R3 | Defines auth/recipient data consumed across repos | Schema, derived paths, active/staged recipient coverage |
| `archetypes` consumer | R3 | Generates runtime credential and request behavior | No store/persistent leak; stored Pi auth cannot override the managed resolver |
| `nexus` command | R3 | Accepts and rewrites encrypted credential state | No token argv/env/output/plaintext file; atomic rollback |
| `deploy` integration | R3 | Couples the three compatible revisions | Lock graph and composed outputs |
| Owner-only live canary | R3 | Touches one scoped credential and one disposable VM | Safe prompt, correct target, managed auth source, real request before cleanup |

## Interface Contracts

### Secrets data

`pi-providers.json` is an object keyed by provider ID. IDs match `^[a-z0-9][a-z0-9-]*$`. Each value has exactly:

- `baseUrl`: an `https://` URL with no userinfo, query, fragment, whitespace, or trailing `/`.
- `api`: `openai-completions` or `openai-responses`, used as the default adapter.
- `models`: a non-empty array with unique non-empty `id` values. Entries may additionally carry `name`, `api`, `reasoning`, and `maxTokens`; a model-level `api` has the same two-value enum and otherwise inherits the provider default.
- `target`: exactly one inventory machine name. It must resolve to `type = "dev"` and `runtime = "libvirt"`.

The provider ID derives every credential name and path: credential `pi-provider-<id>`, ciphertext `secrets/pi-providers/<id>.age`, and guest plaintext `/run/agenix/pi-provider-<id>`. The secrets flake exports `lib.piProviders` and a per-machine projection. The same exported recipient function drives `secrets.nix` and the Nexus command: Nexus host recipients plus every active/staged host key for the one target. Unknown targets, duplicate model IDs, unsupported fields/adapters, a missing ciphertext, or recipient drift fail evaluation.

The public template registry is `{}` and generates no provider credential, secret mapping, or VM target.

### Nexus CLI

```text
pi-provider [--dry-run] add <provider> --url <https-url> --target <dev-vm> (--model <id>[,<id>...] | --models-file <path>) [--api <openai-completions|openai-responses>]
pi-provider [--dry-run] rotate <provider>
```

Repeated `--model` accumulates and comma-separated values are equivalent. `--models-file` is mutually exclusive with `--model` and contains only the model array allowed above; it preserves the existing canary's mixed adapter and reasoning metadata. `add` refuses an existing ID, while `rotate` refuses a missing one and changes only its ciphertext. No token flag, token path, noninteractive bypass, or verification bypass exists.

Before prompting, the command resolves the private secrets checkout, requires it clean, validates all metadata/target/recipient preconditions, and calculates every path it may change. `--dry-run` stops there and prints no private values beyond the provider ID, target, tracked paths, and required follow-up actions.

The live path uses hidden `read -r -s`, rejects empty/multiline input, and accepts only RFC 6750 `b64token` values matching `^[A-Za-z0-9._~+/-]+={0,}$`; this keeps quotes and newlines out of curl's configuration grammar. It keeps the value only in shell memory and probes `<baseUrl>/models` with `curl --config -`; the bearer is sent through curl's private configuration stdin, never argv or environment, and response bodies are discarded. It pipes the same in-memory value directly to `age`, stages only ciphertext and non-secret JSON in a `0700` tmpfs directory, atomically installs both files, restores exact old bytes after any partial failure, unsets the value, and prints commit/push, deploy-lock, rebuild, and credential-free verification commands. The command never commits, pushes, rebuilds, or revokes.

### Dev VM and Pi

For each declared provider targeting a libvirt dev VM, `age.secrets` decrypts the ciphertext to the derived `/run/agenix` path owned by the dev user, group `users`, mode `0400`. Nothing copies it into the home directory, a session environment variable, argv, generated JSON/TypeScript, or the Nix store.

Home Manager installs one managed global Pi extension only when the provider set is non-empty. It registers a complete Pi provider with both OpenAI stream adapters as needed. The provider deliberately omits Pi's side-effect-free `check` shortcut, because path existence cannot prove the credential is usable: both `pi auth check` and a request go through `resolve`, which reads and validates the one fixed runtime path, returns only the `Age-managed VM credential` source label, and ignores any stored `auth.json` credential or `models.json` key for that provider. A missing, empty, or newline-bearing runtime file fails loudly without returning the value. Existing unrelated Pi providers/configuration remain untouched.

An empty provider set renders neither Age secrets nor the extension. A managed-symlink collision warns loudly and leaves the operator-authored target byte-unchanged without aborting the rest of Home Manager activation.

## Agent Gates

- The agent stops here until the owner says `go`; approval authorizes only the public implementation and synthetic validation above.
- The agent cannot access Nexus, the live private secrets/deploy forks, or the provider account. The owner supplies the private URL/model metadata and bearer, commits/pushes private ciphertext/data, updates the private deploy lock, and runs the host-side rebuild.
- Before the live canary, the owner exports the existing provider's non-secret model array for `--models-file`; no agent reads or migrates the old token from Pi's `auth.json`.
- The owner keeps the old Pi auth entry through rebuild and live verification. Only after `pi auth check` names the managed source and a real inference succeeds does the owner remove that one old entry without displaying it. This cleanup is the rollback boundary.
- If the target moves to microVM before implementation, this plan is blocked pending the named credential-delivery issues; the agent does not invent a second transport.

## Acceptance Tests

```sh
# secrets: schema, derived inventory/recipients/files, empty public behavior, and sabotage cases
cd <secrets-worktree>
nix build .#checks.x86_64-linux.credential-inventory .#checks.x86_64-linux.pi-provider-registry

# archetypes: generated extension/age-secret lifecycle plus real Pi requests to a local fixture server
cd <archetypes-worktree>
nix build .#checks.x86_64-linux.pi-provider-integration
nix eval .#nixosConfigurations.allod-dev.config.system.build.toplevel.drvPath

# nexus: syntax/shellcheck, safe-input fixture, failed-probe no-op, atomic-install rollback, dry-run, and packaging
cd <nexus-worktree>
nix build .#checks.x86_64-linux.provisioning-contract

# deploy: compatible pins and composition surface without the multi-machine flake-check memory spike
cd <deploy-worktree>
nix eval .#nixosConfigurations.allod-dev.config.system.build.toplevel.drvPath
nix build .#checks.x86_64-linux.composed-layer
```

The archetypes witness must start Pi with a synthetic conflicting `auth.json` credential and a fixture runtime file, then observe the fixture token—not the stored token—in `Authorization: Bearer ...` requests for one Completions and one Responses model. It also proves the auth-status label, exact runtime ownership/mode, missing/empty/multiline failures, no token in generated/store artifacts, no-provider absence, activation collision behavior, and microVM/privacy rejection. Each validator has a sabotage that proves it can fail.

The Nexus witness inspects child argv/environment and every written regular file, fails if fixture plaintext appears outside the private curl stdin/Age pipeline, and compares pre/post bytes after rejected input, failed HTTP verification, failed encryption, and each simulated partial install.

No private value or private build receipt is recorded in this public plan. Canary completion is a credential-free issue comment from the owner stating that dry-run matched the intended provider/target, enrollment and private pin updates completed, the VM rebuilt, `pi auth check` reported the managed source without `--credentials`, both adapter families remained selectable, a real prompt succeeded, and the old local auth entry was retired.

## Rollback Plan

Before live deployment, revert the affected public PR or keep the deploy pins unchanged. After `pi-provider add`/`rotate` but before rebuild, restore the private registry and ciphertext from git; the command's own failure paths already restore exact pre-run bytes.

After rebuild but before old-auth retirement, pin the previous public/private revisions and rebuild; the pre-existing Pi auth entry remains the working fallback, and the new ciphertext can remain inert or be reverted from git. No provider-side token was changed or revoked.

After old-auth retirement, first restore the previous deploy pins and rebuild, then re-enter the same token through Pi's hidden login prompt (or mint a replacement at the provider) before declaring rollback complete. The Age ciphertext remains recoverable authoritative material; rollback never requires printing or exporting it. If activation hit a managed-extension collision, fix/remove only that collision and rebuild—the activation leaves its bytes unchanged.
