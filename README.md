# strategy

Planning and project direction for Allod: active development plans, the review
prompts that harden them, rough brainstorm notes, and archived material from
work that has shipped or been superseded. Everything here is Markdown — this is
a documentation repo, not code.

`strategy` holds time-bound intent: what to build next, how a specific change
should be reviewed, and where the project is heading. It is distinct from
`memory`, which holds the durable workflow conventions and gotchas agents follow
on every task. Plans here are written to the dev-plan format and review process
that `memory` defines; once a plan is implemented or superseded it moves to
`archive/`.

## Layout

```
dev-plans/            active development plans, one file per effort
review-prompts/       active review prompts that iterate on those plans
brainstorm/           rough exploratory notes and project-direction drafts
archive/              retired material, kept for history
  dev-plans/          plans whose work has merged or been superseded
  review-prompts/     the review prompts that drove those plans
  user-stories/       requirements analysis that predated a plan
```

## What each area holds

- **`dev-plans/`** — structured implementation plans, one per unit of work. Each
  opens with a tracking issue and follows the dev-plan format defined in
  `memory` (Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates,
  Acceptance Tests, Rollback Plan), and maps to one or more PRs across the code
  repos (`tools`, `nexus`, `secrets`, `profiles`).
- **`review-prompts/`** — adversarial agent prompts that drive multi-pass review
  of a plan. Each defines a reviewer persona, focus areas, and a severity rubric
  (`BLOCKER` / `GAP` / `SIMPLIFY` / `QUESTION`), and ends by committing plan
  fixes and updating its own focus areas for the next pass. Most pair
  one-to-one with a plan in `dev-plans/`.
- **`brainstorm/`** — rough notes not yet promoted to a plan: proposed future
  VMs and capabilities (a Bitcoin service VM, a Whonix-style Tor gateway,
  multi-instance VM naming), boundary and design discussions, and
  project-direction drafts such as `licensing.md` and the `org-profile.md`
  landing-page draft.
- **`archive/`** — retired versions of the three categories above, plus a
  `user-stories/` requirements analysis, kept so the history of a decision stays
  readable.

## This repo owns

- active and archived development plans
- the review prompts that harden those plans
- brainstorm notes and project-direction drafts

## This repo does **not** own

- durable workflow conventions, gotchas, and the plan and review formats
  themselves (`memory`)
- code, VM modules, and provisioning (`tools`, `vm`, `nexus`, `profiles`)
- identities, machine specs, and secrets (`secrets`, `inventory`)

## Lifecycle

A plan and its review prompt sit in the active directories while the work is in
flight. When the work merges or the plan is superseded, both move to the
matching `archive/` subdirectory. Brainstorm notes are pre-plan: they either
graduate into a dev plan (or a durable `memory` doc) or remain as history.

## Related repos

- `memory` — durable workflow memory; defines the dev-plan format and review
  process the plans here follow
- `tools`, `nexus`, `profiles`, `secrets`, `vm`, `inventory` — the code and data
  repos whose changes these plans direct and whose issues the plans track

## Cloning

    git clone https://forge.anarch.diy/allod/strategy.git
