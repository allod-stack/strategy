# Phone-First Agent Service Brainstorm

## Context

Goal: decide how to test an OpenClaw- or Hermes-like service that can be
controlled from a phone and can handle selected Allod tasks semi-autonomously,
especially creating a new VPS from an Allod profile after explicit owner
approval.

This is a testing-the-waters note, not an implementation plan. The default
posture should be: keep the mobile client boring, keep the server narrow, and
only add agent autonomy where the Allod stack has a clear capability boundary.

Architectural baseline from `allod/memory/architecture.md`:

- Allod is a self-sovereign NixOS VM stack for agentic coding and privacy work.
- Privacy and secret boundaries outrank everything else.
- Human authority over irreversible and trust-changing acts is structural.
- Third parties are integrations, never foundations.
- The stack is single-user; the owner is the root of trust.
- Code, data, and keys have different owners: inventory owns machine facts,
  secrets own identity and trust roots, framework repos own behavior.
- Privilege flows downhill: human, host, VM, agent.
- Agents never merge, sign, provision, or publish.
- Boundaries must be enforced by capability absence, cryptography, topology, and
  server-side permissions before relying on prompts or prose.

## Working Principles

For this project, security, privacy, and usability are the priority order. That
fits the architecture doc as: protect privacy and secrets first, preserve human
authority second, then move forward with the least ceremony that still enforces
the boundary.

The simplest useful thing is not a full mobile app. It is a small private web
control surface with enough ergonomics to use from iOS and GrapheneOS:

- submit a task
- watch status
- approve or reject sensitive steps
- interrupt a running task
- inspect logs after the fact

The agent service should start as a control plane and supervisor, not as a
general shell-over-chat system.

The biggest architecture tension is the word "autonomous." In current Allod,
host-side provisioning requires the human at the terminal, and agents never
provision. A phone-controlled service can fit the principles only if it is
treated as a narrow owner-operated daemon with explicit human approvals, not as
an agent that has gained host authority. If this becomes more than a local
experiment, it needs a normal dev plan and an explicit owner ruling on whether
"human at the phone" can satisfy the host-authority gate for specific
allowlisted workflows.

## What "Autonomous" Should Mean First

For Allod, autonomy should mean the service can safely execute a small set of
predefined workflows after the owner approves the trust-changing steps:

- create a provider-side VPS
- wait for SSH
- run a NixOS install from an Allod profile
- enroll the machine into the private access network
- run verification
- report the final endpoint and residual risk

It should not initially mean:

- arbitrary shell commands from a phone prompt
- unconstrained OpenClaw/Hermes tool access to the host
- any agent approving or initiating its own provisioning authority
- direct access to private keys or plaintext provider tokens
- public webhooks that can trigger provisioning
- agent self-modification of the provisioning workflow

## MVP Shape

Recommended first experiment:

```text
iPhone / GrapheneOS phone
  |
  | private network only
  v
Allod operator web UI
  |
  | local job queue
  v
Allod operator daemon
  |
  +-- narrow provisioning workflow runner
  +-- optional agent worker for summarization/planning
  +-- audit log and task state in SQLite
```

Properties:

- No public ingress for the first test.
- A PWA-like web UI works on iOS and GrapheneOS without app-store work.
- The server binds only to a private interface.
- Authentication is layered: network admission plus device-bound app auth.
- Task execution goes through allowlisted workflows, not arbitrary commands.
- High-risk steps require explicit approval from the phone.
- OpenClaw/Hermes workers, if used, cannot approve or execute privileged
  provisioning actions directly.
- Logs are retained locally, with secret redaction by design.

## Mobile Client Options

### 1. Private Web UI / PWA

This should be the first path.

Pros:

- One client for Apple and GrapheneOS.
- No app store, signing, mobile build chain, or native dependency churn.
- Works with a small server-rendered UI.
- Can be installed to the home screen on both platforms.
- Can later add Web Push, badges, and offline shell caching if worth it.

Risks:

- Push support differs by platform and browser.
- iOS Web Push requires the app to be added to the Home Screen.
- GrapheneOS push behavior depends on browser and Google Play choices; do not
  make push a hard requirement.
- File sharing, voice, background execution, and secure local storage are weaker
  than native apps.

MVP stance:

- Do not depend on push.
- Start with manual refresh plus server-sent events while the page is open.
- Add Web Push later only for low-detail notifications such as "task needs
  approval".
- Keep notification payloads metadata-light.

### 2. Native App Later

Only consider native once the web UI hits a real wall:

- reliable push becomes mandatory
- share-sheet integration matters
- local encrypted storage needs are substantial
- voice capture becomes central
- browser WebAuthn/PWA support is too inconsistent

Candidate stacks to research later:

- Swift + Kotlin: best platform fit, highest maintenance.
- Flutter: one codebase, still a mobile app project.
- Tauri mobile: attractive if the server/UI is already Rust/web, but verify iOS
  and Android maturity before committing.
- Capacitor: simple wrapper around the web UI, but adds a mobile toolchain while
  retaining many web limitations.

Native is probably a phase-3 decision, not a phase-1 decision.

### 3. Messaging Channel

OpenClaw and Hermes both emphasize gateway-style messaging integrations. This
is useful, but it should not be the initial control surface for privileged Allod
tasks.

Messaging is best as a notification or low-risk command layer after the
capability model is tested in the web UI.

## Access Network

### Raw WireGuard

Best security/privacy baseline:

- simple cryptographic identity
- no third-party coordination service
- very small conceptual model
- mobile clients exist on iOS and Android

Costs:

- manual key enrollment and revocation
- dynamic endpoint and roaming handling is more hands-on
- less polished for "new phone, enroll quickly"

Good direction if this becomes durable infrastructure.

### Tailscale

Best first-test usability:

- polished iOS and Android clients
- MagicDNS and ACLs reduce custom plumbing
- easy server enrollment
- Tailnet Lock can reduce trust in the coordination server after setup

Costs:

- SaaS control plane by default
- account/identity provider dependency
- more policy surface than raw WireGuard

MVP stance:

- Tailscale is acceptable for the experiment if the service is still protected
  by app-level auth and no provider tokens are exposed to the client.
- Do not use Funnel for the privileged control plane. Funnel deliberately exposes
  a service to the broader internet. Use tailnet-only access instead.
- Revisit raw WireGuard or Headscale if the experiment becomes permanent.

### Public HTTPS

Usable, but not the first choice.

If public access is required later:

- terminate with Caddy or nginx
- require WebAuthn/passkeys or mTLS
- keep the API small and rate-limited
- add request logging and alerting
- use separate approval for destructive actions
- do not expose raw agent or MCP endpoints

### Tor Onion Service

Interesting for privacy, but probably not the first usability path:

- avoids public DNS/IP exposure
- works without a VPN account
- weak phone UX compared with normal browser/PWA use
- no practical push-notification story

Research later if the privacy model needs location hiding more than mobile
ergonomics.

## Application Authentication

Use at least two layers:

1. Network membership: WireGuard/Tailscale device admission.
2. Application auth: passkey/WebAuthn or mTLS client certificate.

Passkeys are attractive for a phone-first web UI because they avoid shared
passwords and are phishing-resistant when bound to the correct origin. The
practical questions are recovery, device migration, and whether the service has
a stable HTTPS origin inside the private network.

mTLS is attractive for raw security and simple server enforcement, but mobile
certificate installation and renewal can be annoying. It may fit better as an
operator-only fallback than as the main UX.

Avoid:

- bearer tokens in URLs
- API keys copied into mobile notes/password fields
- long-lived shared passwords
- "send the magic link to chat" flows for privileged approval

## Server Stack

### Go + SQLite + systemd

Likely best fit for a small Allod operator service.

Pros:

- static single binary is easy to package in Nix.
- small runtime and dependency surface.
- good standard library for HTTP, templates, TLS, and process supervision glue.
- SQLite is enough for task state, logs, approvals, and idempotency keys.
- systemd units can isolate the web daemon and task workers.

Cons:

- less rapid UI iteration than full-stack TypeScript.
- WebAuthn implementation should use a mature library, not homegrown crypto.

This is the strongest default if building a custom service.

### Python / FastAPI

Good for prototyping, less ideal as permanent privileged infrastructure.

Pros:

- fast iteration
- good ecosystem for agent glue
- Hermes itself is Python-heavy

Cons:

- larger dependency/runtime surface
- packaging and supply-chain review are more work
- easier to accidentally grow into an unbounded automation server

### TypeScript / Node

Best alignment with OpenClaw-style ecosystems.

Pros:

- good web/PWA ergonomics
- OpenClaw is already Node-based
- many gateway and real-time UI libraries

Cons:

- npm dependency surface is large
- Node runtime is heavier than a small Go daemon
- privileged infrastructure code needs stricter dependency discipline

This is better if the decision is to extend OpenClaw directly instead of writing
an Allod-specific operator.

### Rust

Good security posture, slower first experiment.

Pros:

- strong single-binary story
- good for long-lived infrastructure
- memory safety and explicit error handling

Cons:

- slower to iterate
- web UI and WebAuthn integration may cost more time

Rust is worth revisiting if the service becomes a durable component.

## Agent Harness Choices

### Tiny Allod Operator First

Build the smallest service that can:

- create jobs
- run fixed workflows
- ask for approval
- stream logs
- interrupt jobs
- call an optional agent for planning or summarization

This avoids adopting a broad agent framework before the security boundary is
clear. The "agent" can initially be a tool behind the daemon, not the daemon
itself.

### OpenClaw

Useful traits:

- self-hosted gateway model
- broad channel support
- local/personal assistant framing
- already thinks in terms of phone-accessible channels

Concerns:

- broad channel surface is the opposite of the first-test security posture.
- Node/package surface needs review before granting host capabilities.
- Its docs warn that connecting a channel can put the agent in position to run
  commands, read/write workspace files, and send messages outward.

Research direction:

- Can OpenClaw run as an unprivileged worker behind an Allod operator daemon?
- Can it be restricted to a single workspace and a tiny tool set?
- Can all channel access be disabled while keeping its agent loop?

### Hermes

Useful traits:

- explicitly designed to live on a VPS and be reached from messaging platforms.
- built-in memory, skills, cron, and multiple terminal backends.
- mature "always running agent" mental model.

Concerns:

- self-improving skills and persistent memory are powerful but increase review
  burden.
- broad tool surface is risky for provisioning until policy is proven.
- Telegram-style access is convenient but not the desired security/privacy
  baseline for privileged tasks.

Research direction:

- Can Hermes be run with only a local/SSH backend and no public messaging bridge?
- Can approval policy be enforced outside Hermes?
- Can skills/memory be disabled or scoped for provisioning workflows?

## VPS Provisioning Automation

Allod already has the core NixOS installation idea:

- declarative profiles
- inventory-owned machine/platform data
- disko for declarative disk layout
- nixos-anywhere for remote SSH installation
- age/agenix for secrets
- verification scripts

The missing cloud piece is provider-side lifecycle:

```text
profile name
  -> provider server spec
  -> create VPS with temporary SSH access
  -> wait for SSH
  -> nixos-anywhere install profile
  -> enroll private network
  -> verify
  -> record inventory/known-host state
```

Creating a VPS, changing bootstrap access, installing an OS profile, and
enrolling private networking are trust-changing operations. The service can
prepare and execute them, but the approved design should keep the owner approval
as a structural gate.

### Simplest First Test

Do this manually once:

1. Create a VPS in the provider console or CLI.
2. Add temporary bootstrap SSH access.
3. Run a one-off `nixos-anywhere` install from the intended Allod profile.
4. Confirm the generated profile works outside libvirt assumptions.
5. Document the exact manual cloud fields that would need automation.

This validates the profile and disk model before building a cloud orchestrator.

### Small Automation

Add an `allod vps create <profile>` or `allod-operator provision <profile>`
workflow once the manual path works.

Responsibilities:

- read provider/project/region/size/image from private inventory or profile
  metadata
- create the VPS through a narrow provider adapter
- inject only a short-lived bootstrap SSH key
- never print provider tokens
- store no plaintext provider token in task logs
- wait for SSH and host key initialization
- call `nixos-anywhere`
- run verification
- disable or remove bootstrap-only access

### Provider API vs OpenTofu

Provider CLI/API:

- simplest for one VPS provider
- easy to wrap in Go
- less state machinery
- better for testing the shape of the workflow

OpenTofu:

- better once cloud resources grow beyond one server
- declarative infrastructure state
- supports encrypted state and plan files
- adds state lifecycle, locking, and provider dependency concerns

MVP stance:

- Start with the provider's CLI/API for one provider.
- Keep the interface shaped so it can move to OpenTofu later.
- Use OpenTofu when there are multiple resources, persistent networks,
  firewalls, volumes, or a need to diff planned cloud changes before apply.

### Provider Selection

Hetzner Cloud is a practical first provider to research because:

- nixos-anywhere docs use a Hetzner cloud machine in their quickstart.
- Hetzner has an official Cloud API and `hcloud` CLI.
- OpenTofu/Terraform provider support exists.

Do not bake Hetzner into public profiles. Provider facts belong in private
inventory or a provider adapter boundary.

## Integration Technology Ranking

For the first privileged operator channel:

1. Private web UI over WireGuard/Tailscale.
2. Matrix direct message to a self-hosted homeserver, only after E2EE bot/device
   handling is understood.
3. SimpleX, if metadata privacy matters enough to pay integration complexity.
4. Signal, if an unofficial linked-device bridge is acceptable and reliable.
5. Telegram/WhatsApp/Discord only for low-sensitivity notifications.

Reasoning:

- Web UI avoids third-party message retention and bot-platform limitations.
- Matrix is self-hostable and has real E2EE, but encrypted bot workflows and
  device verification add operational complexity.
- SimpleX has the strongest metadata story, but integration maturity needs
  validation.
- Signal has excellent protocol properties, but automation usually relies on
  unofficial bridges and phone-number/account constraints.
- Telegram Secret Chats are one-to-one device-bound E2E, but bot workflows are
  not Secret Chats. Treat Telegram bots as convenience, not privacy.

## Capability Model

The important design choice is not the UI framework. It is what the service is
allowed to do.

Start with explicit capabilities:

- `plan-vps`: produce a proposed server/profile/provider plan.
- `create-vps`: call provider API after approval.
- `install-profile`: run `nixos-anywhere` for a named profile after approval.
- `verify-vps`: run read-only verification.
- `destroy-vps`: require a stronger confirmation phrase or out-of-band approval.
- `rotate-bootstrap-key`: narrow secret/key lifecycle action.

Each capability should declare:

- allowed user role or device
- required approval level
- command runner identity
- allowed environment variables
- secret inputs
- log redaction rules
- timeout
- rollback or cleanup behavior

Avoid a generic `run-command` capability for the first version.

## Approval UX

The phone UI should show a compact approval card:

- task id
- requested action
- target provider/project/profile
- resources to be created or destroyed
- exact high-level command, with secrets redacted
- estimated cost if available
- risk label
- approve, reject, request changes, stop task

Approval should be idempotent and auditable. The daemon should record who/what
approved, when, from which authenticated device, and the immutable task version
that was approved.

## Secret Handling

Apply the existing Allod rules:

- no token-print, token-export, or token-display commands
- no tokens in argv
- no tokens in URLs
- no plaintext provider tokens in task logs
- narrow credential helpers instead of generic secret-dump commands
- age-encrypted secrets remain the source of truth

Provider token options to research:

- short-lived provider API tokens, if the provider supports them
- scoped project tokens with only server lifecycle permissions
- one token per provider adapter
- token passed to the runner through a private fd or root-owned environment file
- agenix secret mounted only for the daemon user that needs it

## Deployment Shape

Possible placement options:

### Run on Nexus

Pros:

- matches current host-side provisioning reality
- can call existing host scripts directly
- fewer distributed secret edges

Cons:

- exposes a remote control surface on the host
- needs careful systemd hardening and network binding
- must not become arbitrary host automation

This may be the most realistic first internal test if the goal is to automate
existing host-only provisioning. It is also the option that most directly
changes the current "human at the terminal" rule, so it needs the clearest dev
plan and owner ruling before becoming normal workflow.

### Run on a Service VPS

Pros:

- closer to Hermes-style always-on cloud agent
- can create other VPS instances from anywhere
- isolates the operator service from the home host

Cons:

- needs cloud provider credentials on the service VPS
- must access private Allod inputs safely
- may not be able to perform host-only Nexus operations

This is the right target if "VPS from profile" becomes cloud-native rather than
Nexus/libvirt-native.

### Run in a Dedicated Local Service VM

Pros:

- isolates from Nexus more than a host service
- keeps secrets and network local
- can be rebuilt with existing Allod machinery

Cons:

- still depends on local host availability
- may need explicit bridge access to host-only provisioning commands

This is a good compromise for testing daemon hardening before putting provider
tokens on a public VPS.

## Hardening Notes

For the daemon:

- bind to private interface only
- run as a dedicated user
- use systemd sandboxing
- separate web process from worker process if that stays simple
- make the worker execute allowlisted commands only
- store state in SQLite with restrictive permissions
- keep logs local and redact at the source
- add a kill switch in the UI
- enforce per-task timeouts
- keep a clear emergency manual recovery path

For workers:

- prefer `systemd-run` transient units or dedicated services per job
- set CPU/memory/runtime limits
- use temporary working directories
- pass secrets through files/fds with restrictive permissions, not argv
- mark artifacts as public/private before exposing them in the UI

For agent harnesses:

- run OpenClaw/Hermes as unprivileged workers, if used at all
- disable broad channel bridges for privileged workflows
- expose only the Allod operator API to the harness
- keep the harness unable to approve its own privileged actions

## Research Questions

Immediate:

- Can an installable PWA on iOS and GrapheneOS handle the basic task UI well
  enough without push?
- Does passkey/WebAuthn work reliably against a private-network HTTPS origin?
- Is Tailscale acceptable for the first test, or should raw WireGuard be the
  first implementation despite worse enrollment UX?
- Which VPS provider should be the first adapter?
- Can a current Allod profile install cleanly on that provider with
  nixos-anywhere and disko?

Provisioning:

- What profile metadata is missing for cloud VPS creation?
- Where should provider/project/region/size live in private inventory?
- How should bootstrap SSH keys be generated, stored, rotated, and removed?
- How do we record host keys without leaking private material?
- Can provider firewalls default to private-network-only management?

Agent integration:

- Is a custom tiny Allod operator enough for phase 1?
- Can Hermes/OpenClaw be safely restricted to a non-approving worker role?
- Is MCP useful internally for tool boundaries, or does it add too much surface
  before the workflow is stable?

Mobile notifications:

- Is no-push acceptable for the first test?
- Can Web Push payloads be made metadata-light enough?
- Does GrapheneOS require sandboxed Google Play for the chosen browser/PWA push
  behavior?
- Would Matrix/SimpleX notifications be a better later notification channel than
  web push?

## Suggested First Milestone

Write no general autonomous-agent framework yet.

Build or prototype:

1. One private web page reachable only over Tailscale or WireGuard.
2. One login method, preferably passkey if private HTTPS is straightforward.
3. One SQLite-backed job table.
4. One fake provisioning job that streams logs and requires approval.
5. One real read-only verification job.
6. One manually created VPS installed with an Allod profile using
   `nixos-anywhere`.

After that, decide whether to:

- write the dev plan and owner ruling for phone-approved provisioning
- automate provider creation with a small provider adapter
- introduce OpenTofu state
- add Web Push
- run Hermes/OpenClaw as a worker
- move from local service to service VPS

## References

- OpenClaw README: https://github.com/openclaw/openclaw
- OpenClaw personal assistant safety notes: https://docs.openclaw.ai/start/openclaw
- Hermes README: https://github.com/nousresearch/hermes-agent
- Hermes docs: https://hermes-agent.nousresearch.com/docs/
- WebKit iOS/iPadOS Web Push: https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
- GrapheneOS usage guide: https://grapheneos.org/usage
- WireGuard overview: https://www.wireguard.com/
- Tailscale Funnel docs: https://tailscale.com/docs/features/tailscale-funnel
- Tailscale Tailnet Lock docs: https://tailscale.com/docs/features/tailnet-lock
- WebAuthn Level 3 spec: https://www.w3.org/TR/webauthn-3/
- nixos-anywhere quickstart: https://nix-community.github.io/nixos-anywhere/quickstart.html
- disko README: https://github.com/nix-community/disko
- OpenTofu state encryption: https://opentofu.org/docs/language/state/encryption/
- Hetzner Cloud API overview: https://docs.hetzner.cloud/
- Signal protocol docs: https://signal.org/docs/
- Matrix E2EE guide: https://matrix.org/docs/matrix-concepts/end-to-end-encryption/
- SimpleX privacy notes: https://simplex.chat/privacy/
- Telegram Secret Chats docs: https://core.telegram.org/api/end-to-end
