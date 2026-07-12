# Add pi Memory Adapter

## Tracking Issue

TBD — create in `allod/memory` before the implementation PR.

## Context

This is the public half of adding the pi coding agent to the dev-VM stack: pi's
memory adapter. It mirrors the existing `adapters/claude/CLAUDE.md` and
`adapters/codex/AGENTS.md`. The dev-VM home-manager wiring that installs pi and
generates its pointer to this file is a separate integration effort; this plan
produces only the adapter and is self-contained within `allod/memory`. A public
agent can complete it without access to any private repository.

## Goal

Add `adapters/pi/AGENTS.md` to `allod/memory` so pi has its own memory-read
adapter, symmetric with the claude and codex adapters.

## Scope

In scope:

- `allod/memory`: new `adapters/pi/AGENTS.md`, mirroring `adapters/codex/AGENTS.md`.

Out of scope:

- Any dev-VM, home-manager, or profiles wiring (separate integration plan).
- The `agent-memory` copy of this adapter (added when pi rolls out to VMs whose
  memory checkouts include `agent-memory`).
- Changing the existing claude or codex adapters.

## Risk Assessment

Residual risk: R0 Docs/metadata. A new documentation file with no runtime,
generated, operational, or state behavior in this repo; rollback is deletion.

Human scrutiny:

- Confirm the adapter matches the sibling adapters' structure and points readers
  at the repo's `memory.md`.

## Interface Contracts

- `adapters/pi/AGENTS.md` mirrors `adapters/codex/AGENTS.md`: a `# Pi Adapter`
  heading followed by the same "Read `../../memory.md` relative to this adapter
  file before relying on Allod workflow memory." instruction. It is pi's own
  file, not a symlink or include of the codex adapter. A pi-specific policy
  section may be added if one is needed; none is required for parity with codex.
- The filename must be `AGENTS.md` under `adapters/pi/`, because pi reads
  `AGENTS.md` with the same semantics as codex.

## Agent Gates

- None. `allod/memory` is public; a public agent with write access can complete
  and push this without seeing any private repository.

## Acceptance Tests

```sh
cd <allod/memory checkout>
test -f adapters/pi/AGENTS.md
head -1 adapters/pi/AGENTS.md | grep -q '# Pi Adapter'
grep -q 'memory.md' adapters/pi/AGENTS.md
```

## Rollback Plan

Delete `adapters/pi/AGENTS.md`. Nothing else in this repo depends on it.
