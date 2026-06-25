# Agent Harness: Open-Source Alternatives

Evaluate open-source, model-agnostic coding agent harnesses as potential
replacements for proprietary agents in the dev VM stack.

## Why

- Drop `allowUnfree` from dev VM home-manager configs
- Single harness instead of maintaining two (Claude Code + Codex)
- Model flexibility without changing tools or NixOS config
- Local model capability for offline/private work

## Candidates

- [pi](https://github.com/earendil-works/pi) -- MIT, minimalist 4-tool core,
  50+ extension hooks, nixpkgs package available

## Key Questions

- Can the shared memory architecture be replicated via extensions?
- What replaces plan mode?
- Is the migration cost bounded enough to justify the switch?

## Status

**Not implemented.** The evaluation was done but no migration happened.
Revisit when the stack is stable post-open-source.
