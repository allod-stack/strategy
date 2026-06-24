# Secrets vs Preferences Boundaries

## Context

The home-shared decomposition moved private desktop preferences out of
`profiles/modules/home-shared.nix` and into `secrets.homeModules.preferences`.
That solved the immediate publication boundary, but it also exposed a naming and
ownership question: `secrets` now contains non-secret personal preferences.

The concrete example is the `claude` and `codex` shell aliases. They are
personal workflow glue, but they are only used on dev VMs and do not contain
private values. That makes them a poor fit for `secrets.homeModules.preferences`
and a better fit for `profiles/hosts/dev/home-shared.nix`.

## Current Boundary Guidance

Use `profiles/modules/` for public framework modules. These should be reusable,
parameterized, and free of personal values.

Use `profiles/hosts/dev/home-shared.nix` for public dev-VM workflow defaults.
This is the right home for non-secret agent tooling behavior that applies only
to dev VMs, including shell aliases such as `claude` and `codex`.

Use `secrets.homeModules.preferences` for private personal Home Manager
preferences that should not live in the public profiles repo. This includes
browser, editor, shell, and desktop choices when the values are personal or when
the preference module depends on private inputs.

Use `secrets` for credential and identity authority: identity data, encrypted
tokens, key inventory, credential rotation state, and validation checks.

## If Secrets Split Later

If `secrets` is split into two private repos, a likely boundary is:

`secrets`:

- `identity.nix`
- `credentials.nix`
- `secrets/*.age`
- key inventory and public keys
- credential rotation scripts
- validation checks for identity, credentials, and encrypted files
- git policy files when they are part of access or security policy

`preferences`:

- Home Manager preference modules
- private browser, editor, shell, and desktop defaults
- private flake inputs needed only by preferences, such as `nvim-config` or NUR
- machine-independent personal UX choices
- preference-specific checks and documentation

The split should not happen just because one preference module exists. The cost
is another flake input, another lock file edge, another PR sequence, and another
repo that must be kept in sync during VM rebuild work.

## Complexity Threshold

Keep preferences in `secrets` while the only cost is a slightly broader repo
purpose and the module remains small, private, and easy to review.

Prefer a separate `preferences` repo when one or more of these becomes true:

- Preference churn regularly touches the same repo as credential rotation.
- Different people or agents should access preferences without access to the
  credential inventory.
- Preferences grow into multiple modules, such as `editor.nix`, `browser.nix`,
  `desktop.nix`, or `agents.nix`.
- Preference inputs create frequent lock churn unrelated to secrets.
- Preference checks, docs, and review criteria become distinct from credential
  validation.
- The repo name `secrets` starts hiding the actual ownership model instead of
  clarifying it.

The useful signal is not line count. The signal is coupling pain: if editor,
browser, or agent workflow changes routinely share review, lock, or access
surface with credential work, split the repo.

## Promotion To Architecture

Keep this note in `brainstorm/` while the boundaries are still exploratory and
the immediate actions are small cleanups.

Promote the settled model into a new `architecture/` directory when this line of
thinking becomes a standing reference instead of a one-off discussion. Good
promotion triggers:

- A `preferences` repo is actually introduced.
- More than one private Home Manager module exists.
- PR reviews start relying on this boundary to accept or reject changes.
- Another decomposition plan needs a stable source of truth for repo ownership.
- Agent instructions need to cite a durable rule for where config belongs.

Good promoted document names:

- `architecture/private-repos.md`
- `architecture/config-ownership.md`
- `architecture/personal-preferences.md`

The promoted architecture doc should describe the current accepted shape, an
ownership matrix, and examples. This brainstorm note can remain as the history
of tradeoffs and threshold thinking.
