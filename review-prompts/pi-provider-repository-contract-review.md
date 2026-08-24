# Review the pi-provider repository contract plan

Review [the plan](../dev-plans/pi-provider-repository-contract.md) against [allod/nexus#29](https://forge.anarch.diy/allod/nexus/issues/29) and the current public `allod/secrets`, `allod/nexus`, `allod/profiles`, and `allod/inventory` repositories. This is a public-only review: do not request, inspect, infer, decrypt, or reproduce private provider data. Return only actionable BLOCKER, GAP, or SIMPLIFY findings, each tied to the plan contract and executable evidence; if there are none, say so plainly.

## Pass Budget

Residual risk is R3: at most two plan-review passes. A claim an existing fixture or Nix evaluation can execute must be settled by that witness rather than another prose pass.

## Focus Areas

1. Does tmpfs staging exercise each redirected repo's actual output binding, including a static/disconnected `secrets.nix` negative case, without creating a second recipient source of truth?
2. Can the existing secrets derivation represent a split hypervisor registry, a nonliteral hypervisor name, active/staged rotation, exact order, and global duplicate rejection without requiring the hypervisor in `machine-host-keys.json`?
3. Does the Nexus preflight finish before dry-run completion and bearer input, and does post-install equality force the mutated consumer outputs without hardcoded template check names?
4. Does `inventoryCheckout` reach `pi-provider` as `INVENTORY`, and does strict alias resolution select redirected working trees without silently falling back after malformed data?
5. Do copied private sources remain confined to the existing tmpfs lifecycle, and do dry-run, rollback, interruption recovery, and public/private boundary claims remain true?

## Review Evidence

- **Pass 1** — `gpt-5.6-sol` (Codex runner), reasoning `high`, reviewed the initial uncommitted plan. 1 BLOCKER, 1 GAP, 1 SIMPLIFY. BLOCKER: callable-output evaluation did not prove redirected registry files fed the outputs before bearer input; fixed by replacing the proposed cross-repo function with behavioral evaluation of staged tmpfs repo copies and exact staged/live equality, plus a disconnected-output negative witness. GAP: duplicate scope was ambiguous; fixed by retaining global rejection, prohibiting silent deduplication, defining order as the stored output list, and naming the duplicate classes. SIMPLIFY: two slices were called independently landable despite a dependency; the first attempted fix removed the secrets slice.

- **Pass 2** — `gpt-5.5` (Codex runner), reasoning `high`, reviewed the revised uncommitted plan. 1 BLOCKER. The public secrets derivation still required the hypervisor in `machineHostKeys`, so Nexus behavioral staging alone could not represent the issue's split registry. Fixed by restoring an ordered secrets slice: the live derivation now receives `identity.hostPublicKeys` separately, VM keys remain in `machineHostKeys`, and the shared constructor retains a measured compatibility fallback for deploy. The tmpfs, prompt-ordering, alias, and rollback contracts had no further findings. The two-pass R3 budget is exhausted; executable secrets, deploy-consumer, and Nexus sabotage witnesses settle the fix.
