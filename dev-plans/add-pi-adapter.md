# Add Pi Coding Agent to Dev VMs

## Tracking Issue

`allod/profiles#4` - https://forge.anarch.diy/allod/profiles/issues/4. Adapter PRs
use `Refs allod/profiles#4`; the final `allod/profiles` PR carries
`Closes allod/profiles#4`.

## Context

The earlier version of this plan covered only the public `allod/memory` adapter.
The rollout needs the whole dev-VM integration: install the packaged Pi coding
agent, generate Pi's global memory pointer, and keep provider credentials
operator-supplied at runtime instead of committing or provisioning them.

The pinned `nixpkgs-unstable` input already exposes `pkgs.pi-coding-agent`
version `0.80.3`; its `meta.mainProgram` is `pi`. The Nix wrapper sets
`PI_SKIP_VERSION_CHECK=1` and `PI_TELEMETRY=0` by default.

Pi discovers global context from `~/.pi/agent/AGENTS.md`, then also loads
project ancestor `AGENTS.md` or `CLAUDE.md` files unless `--no-context-files` is
used. The existing `profiles/modules/ai-agents.nix` module already generates
Claude and Codex memory pointer files from `memoryCheckouts`; Pi should follow
that pattern with `adapters/pi/AGENTS.md`.

## Goal

Install Pi on the dev VM profile and generate its memory pointer so the operator
can run `pi` or `yolo pi` from `~/work` with external API keys supplied at
invocation time.

## Scope

In scope:

- `allod/memory`: add `adapters/pi/AGENTS.md`, mirroring
  `adapters/codex/AGENTS.md`.
- `agent-memory`: add the same `adapters/pi/AGENTS.md` adapter for the private
  memory checkout used by the default dev-VM `memoryCheckouts` list.
- `allod/profiles/modules/ai-agents.nix`: generate
  `~/.pi/agent/AGENTS.md` from `memoryCheckouts`; add a `pi` shell alias that
  starts in `~/work`; add `pi` to the `yolo` wrapper if the final CLI check
  confirms `--approve` remains Pi's project-local resource trust override.
- `allod/profiles/hosts/dev/allod-dev/home.nix`: add
  `pkgs-unstable.pi-coding-agent` beside `claude-code` and `codex`.
- `allod/profiles/flake.nix`: add or update checks that inspect the generated
  Home Manager activation text and package list for Pi.

Out of scope:

- Storing API keys, OAuth data, Pi `auth.json`, Pi `settings.json`, model
  defaults, or provider-specific configuration in any repo.
- Adding age secrets, password-manager integration, or shell helpers that read a
  specific password manager.
- Pinning a Pi model or provider. Operators can use runtime environment
  variables such as `OPENROUTER_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  or `GEMINI_API_KEY`, depending on the provider they choose.
- Updating private dev profile definitions that do not consume the public
  `allod-dev` package list.
- Changing Claude or Codex behavior except for unavoidable shared wrapper/help
  text updates in `yolo`.
- Host rebuild execution. A human runs host-side rebuilds after PRs land.

## Risk Assessment

Residual risk: R2 Medium.

Why:

- The adapter changes are R0 docs/metadata, but the `profiles` change modifies
  generated Home Manager activation behavior, the dev profile package closure,
  and the shared `yolo` wrapper.
- The blast radius is limited to dev VMs. No privacy VM, host provisioning path,
  secret material, forge token, or provider credential is changed.
- Validation can inspect the generated activation text for the exact
  `~/.pi/agent/AGENTS.md` content and can evaluate the allod-dev package list
  for `pi-coding-agent`.
- Rollback is a straight revert plus human rebuild. If a VM already ran the
  activation, the only persistent cleanup is removing generated
  `~/.pi/agent/AGENTS.md` if the operator wants Pi state fully absent.

Human scrutiny:

- Confirm the generated Pi pointer includes every configured memory checkout in
  the same order as Claude and Codex.
- Confirm no API key, provider token, auth file, settings file, or password
  manager path enters the diff.
- Confirm `yolo pi` does not imply stored credentials and only uses Pi CLI flags
  that are present in the packaged version.

No dedicated review prompt is required for this plan as written. The normal
post-PR read-only review pass is enough. Create a review prompt only if the
scope expands into credential storage, provider defaults, package overlays, or
cross-VM rollout beyond the current dev profile.

For multi-PR execution:

| PR or milestone | Risk | Reason | Human scrutiny |
|---|---|---|---|
| `allod/memory` Pi adapter | R0 Docs/metadata | New adapter file only | File name and relative `memory.md` instruction |
| `agent-memory` Pi adapter | R0 Docs/metadata | New private adapter file only | Same shape as public adapter, no duplicated policy drift |
| `allod/profiles` Pi integration | R2 Medium | Home Manager activation, package list, and wrapper behavior | Generated activation, package list, absence of credentials |

## Interface Contracts

- Public adapter path: `allod/memory/adapters/pi/AGENTS.md`.
- Private adapter path: `agent-memory/adapters/pi/AGENTS.md`.
- Adapter content shape:

  ```md
  # Pi Adapter

  Read `../../memory.md` relative to this adapter file before relying on Allod workflow memory.
  ```

- `profiles/modules/ai-agents.nix` generates
  `$HOME/.pi/agent/AGENTS.md` with this structure:

  ```md
  # Pi Adapter
  Read shared instructions from `/home/<username>/work/<checkout>/adapters/pi/AGENTS.md`.
  ```

  There must be one pointer line per `memoryCheckouts` entry, in the same order
  as the generated Claude and Codex pointer files.

- The generated Pi pointer must be written atomically via a temporary file and
  `mv -f`, matching the existing Claude and Codex pointer pattern.
- The normal `pi` alias starts in `~/work` and does not set provider, model, or
  credential environment variables.
- `yolo pi` starts in `~/work` and may pass `--approve` only as Pi's documented
  project-local resource trust override. It must not embed API keys, provider
  choices, model choices, or password-manager commands.
- `profiles/hosts/dev/allod-dev/home.nix` installs
  `pkgs-unstable.pi-coding-agent`.

## Agent Gates

- Create the tracking issue in `allod/profiles` before opening implementation
  PRs, and update this plan's Tracking Issue section.
- If the implementation agent does not have the private `agent-memory` checkout,
  stop before opening the final `allod/profiles` PR and have a human or private
  agent add `agent-memory/adapters/pi/AGENTS.md`.
- Do not create, request, store, or test real provider API keys. Runtime API key
  use is an operator action outside this plan.
- Do not run host-side rebuilds. After the PRs land, the human pulls on the host
  and rebuilds affected dev VMs.

## Acceptance Tests

```sh
cd ~/work/allod/memory
test -f adapters/pi/AGENTS.md
head -1 adapters/pi/AGENTS.md | grep -q '# Pi Adapter'
grep -q 'memory.md' adapters/pi/AGENTS.md

cd ~/work/agent-memory
test -f adapters/pi/AGENTS.md
head -1 adapters/pi/AGENTS.md | grep -q '# Pi Adapter'
grep -q 'memory.md' adapters/pi/AGENTS.md

cd ~/work/allod/profiles
pi_store="$(nix build --extra-experimental-features 'nix-command flakes' --impure \
  --no-link --print-out-paths --expr \
  'let f = builtins.getFlake ("path:" + toString ./.); pkgs = import f.inputs.nixpkgs-unstable { system = "x86_64-linux"; config.allowUnfree = true; }; in pkgs.pi-coding-agent')"
"$pi_store/bin/pi" --version | grep -qE '^[0-9]+[.][0-9]+[.][0-9]+$'
"$pi_store/bin/pi" --help | grep -q -- '--approve'

username="$(nix eval --extra-experimental-features 'nix-command flakes' --raw .#vmFacts.allod-dev.username)"
activation="$(nix eval --extra-experimental-features 'nix-command flakes' --raw ".#nixosConfigurations.allod-dev.config.home-manager.users.${username}.home.activation.llmMemoryLinks.data")"
printf '%s\n' "$activation" | grep -F 'mkdir -p "$HOME/.pi/agent"'
printf '%s\n' "$activation" | grep -F '# Pi Adapter'
printf '%s\n' "$activation" | grep -F "/home/${username}/work/allod/memory/adapters/pi/AGENTS.md"
printf '%s\n' "$activation" | grep -F "/home/${username}/work/agent-memory/adapters/pi/AGENTS.md"
printf '%s\n' "$activation" | grep -F 'mv -f "$HOME/.pi/agent/AGENTS.md.tmp" "$HOME/.pi/agent/AGENTS.md"'

nix eval --extra-experimental-features 'nix-command flakes' --json \
  ".#nixosConfigurations.allod-dev.config.home-manager.users.${username}.home.packages" \
  | jq -e 'any(.[]; contains("pi-coding-agent"))'

nix eval --extra-experimental-features 'nix-command flakes' --raw \
  .#nixosConfigurations.allod-dev.config.system.build.toplevel.drvPath >/dev/null
```

## Rollback Plan

Revert the `allod/profiles`, `allod/memory`, and `agent-memory` commits or PRs.
After the profiles revert lands, a human rebuilds affected dev VMs. If a VM had
already activated the Pi pointer and the operator wants runtime state fully
removed, delete `~/.pi/agent/AGENTS.md` on that VM; no committed secret or
provider credential is created by this plan.
