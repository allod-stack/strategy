# Persistent VM Hostname Status Indicator in Agent Harnesses

## Tracking Issue

`allod/profiles#7` - https://forge.anarch.diy/allod/profiles/issues/7. Single-repo
change; the one `allod/profiles` PR carries `Closes allod/profiles#7`.

## Context

Operators keep several agent sessions open for hours across multiple dev VMs and
cannot tell at a glance which VM a given session is on. A one-time startup
message scrolls away and is useless. The fix is a persistent, always-visible
indicator showing the VM hostname inside each harness's own UI, sourced once from
the host's NixOS config and fanned out per harness.

Two harnesses expose a persistent status surface (verified on `allod-dev`,
`pi` 0.80.3 / `claude-code` 2.1.204 / `codex` 0.142.5 installed):

- **Pi** (primary): a TypeScript extension auto-discovered from
  `~/.pi/agent/extensions/*.ts` sets a persistent footer status via
  `ctx.ui.setStatus(...)` on `session_start`. Modeled on the installed example
  `.../pi-monorepo/examples/extensions/status-line.ts`.
- **Claude Code**: `~/.claude/settings.json` `statusLine` runs a command whose
  stdout becomes a persistent status row above the built-in footer badges. The
  command receives a session JSON object on stdin (schema pinned below); that
  object contains **no** hostname field, so the hostname must be supplied by the
  script itself.

`modules/ai-agents.nix` already generates the per-harness memory pointer files
(`~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`, `~/.pi/agent/AGENTS.md`) atomically
during home activation. The new status-indicator generation lives in the same
module, in a dedicated activation entry, following the same generate-atomically
pattern.

## Goal

Every `pi` and `claude` session on a dev VM shows a persistent `VM: <hostname>`
indicator in its own footer/status line, sourced once from
`osConfig.networking.hostName`. "Persistent" means the normal chat view: Claude's
row is transiently hidden during autocomplete/help/permission prompts and is
suppressed entirely until workspace trust is accepted (Claude docs); Pi's footer
status shows whenever a session is active. Those UI caveats are called out in the
manual verification step, not defects.

## Scope

In scope:

- `allod/profiles/modules/ai-agents.nix`: read the host name from
  `osConfig.networking.hostName`; generate a Claude `statusLine` script (Nix
  store artifact), a Claude settings-merge script (Nix store artifact), a Pi
  status extension (Nix store `.ts`), and a Pi installer script (Nix store
  artifact) that refuses to replace non-managed extension state, all with the
  hostname baked in where relevant; add a dedicated
  `home.activation.agentVmStatus` entry that invokes the Pi installer for
  `~/.pi/agent/extensions/vm-status.ts` and invokes the merge script to set only
  the `.statusLine` key of `~/.claude/settings.json`.
- `allod/profiles/flake.nix`: add an `agent-vm-status` check (sibling to the
  existing `pi-integration` check) that inspects and exercises the generated
  artifacts, including negative/sabotage paths.

Out of scope:

- **Codex**: codex 0.142.5 does expose a `tui.status_line` footer config, but it
  accepts only a fixed set of predefined item identifiers (`spinner`, `project`,
  and model / token-usage / context-usage / branch / approval / current-dir /
  rate-limit items) - there is no custom-text or external-command item, so a
  `VM: <hostname>` string cannot be injected. Excluded until such a hook exists.
  (Assumption: the identifier list contains no custom/command item; the
  implementer should confirm against the full list in one glance.)
- Privacy and hypervisor profiles: `ai-agents.nix` is imported only by the dev-VM
  builder, so those archetypes are untouched by construction.
- Optional extras (cwd / git branch / model / context %): Claude and Pi already
  surface model/cwd/git natively; the missing fact is the hostname, so the MVP is
  hostname-only. Extras are a possible follow-up, not this change.
- Host rebuild execution: a human runs `nixos-rebuild` on the host to make the
  change live (see Agent Gates).
- Pi's own `~/.pi/agent/settings.json`: auto-discovered extensions need no
  settings entry, so this file is not touched.

## Design Decisions

These four decisions are resolved, not left open.

1. **`~/.claude/settings.json` ownership - activation-time `jq` merge of only the
   `.statusLine` key, atomic `mv -f`, guarded by a single-JSON-object preflight.**
   Not `home.file`/full ownership. The live file on `allod-dev` already contains
   operator-authored keys (`model`, `theme`, `tui`,
   `skipDangerousModePermissionPrompt`), and Claude Code itself writes this file
   (`/config`, `/statusline`, etc.). Full ownership would clobber operator- and
   agent-written settings; the authoritative copy of everything except
   `.statusLine` is the operator, not this module (principle 8: the module owns
   only the `.statusLine` fact). Per principle 11 (fail loud, never fall back
   silently) the merge is hardened against every degenerate-but-plausible input:
   the preflight requires the file to slurp to exactly one JSON **object**
   (`jq -e -s 'length==1 and (.[0]|type=="object")'`), which rejects an empty
   file, a bare scalar (`"x"` is valid JSON), a multi-document stream (`{}\n{}`),
   and malformed JSON in one gate; a plain `jq empty` would pass all but the last
   and then silently drop the status line or write a file Claude cannot parse. On
   rejection the script prints a clear error naming the file, touches nothing, and
   exits non-zero. The merge pipeline also owns its own failure
   (`jq ... > tmp || { rm -f tmp; exit 1; }`) so `mv` can never run after a failed
   `jq` regardless of shell options (the merge script runs its own `set -eu`).
   The activation entry invokes the script NON-FATALLY (`${claudeStatusMerge} ||
   true`), so a refusal is loud (the ERROR surfaces in the rebuild output and
   `journalctl -u home-manager-<user>.service`) and the file is left byte-unchanged,
   but it never aborts activation or rewrites the file to `{}`. Non-fatal, not
   unit-failure, is the containment mechanism - see the Activation contract for why
   entry ordering cannot be relied on. The `.statusLine` key is re-asserted
   idempotently every rebuild; all other keys are preserved verbatim.

2. **Hostname source - baked from `osConfig.networking.hostName` at generation,
   for both harnesses.** Not `os.hostname()` / `hostname` at runtime. The feature
   requirement is "sourced once from the host config and fanned out per harness,"
   and principle 8 makes the declared NixOS config the single source of truth for
   the hostname fact. A runtime lookup is a second, independently-drifting source
   and is not the declared config value. Baking a literal also guarantees the Pi
   extension and the Claude script show the identical string.

3. **What the indicator shows - `VM: <hostname>` only (MVP).** This is the actual
   operator need. Extras are out of scope (see Scope).

4. **File placement.** The Pi extension is installed at
   `~/.pi/agent/extensions/vm-status.ts` (the global auto-discovery directory) as
   a symlink to an immutable Nix store `.ts`. It coexists cleanly with the
   existing `~/.pi/agent/AGENTS.md` (different subpath, same module), but the
   installer must also protect the exact `vm-status.ts` path: if that path exists
   and is not a symlink already pointing at a managed `/nix/store/*-pi-vm-status.ts`
   artifact, it prints an ERROR and refuses to replace it. `ln -sfn` alone is not
   acceptable because it overwrites a regular file and, when the destination is a
   directory, succeeds by creating a link inside that directory instead of
   installing the auto-discovered extension path. The Claude status script,
   settings-merge script, Pi extension, and Pi installer script are Nix store
   artifacts (immutable, content-addressed, directly executable in tests).
   Generation lives in `modules/ai-agents.nix` in a new
   `home.activation.agentVmStatus` entry (not folded into `llmMemoryLinks`) whose
   Pi installer and Claude merge are invoked non-fatally (see Interface Contracts)
   so a refusal on operator-authored `vm-status.ts` or malformed operator
   `settings.json` cannot cascade into the other activation entries. A separate
   entry does not isolate the blast radius on its own: home-manager runs every
   entry in one `set -eu` script, so any non-zero exit aborts whatever the DAG
   sorts after it, and that ordering is not controllable (see Interface
   Contracts). Invoking the mutable-state helpers non-fatally is what contains the
   blast radius. That gives the new behavior a contained blast radius, a targeted
   flake check, and an independent rollback.

## Risk Assessment

Residual risk: R2 Medium.

Why:

- Blast radius is dev VMs only (`ai-agents.nix` is imported solely by the dev-VM
  builder). No secret, provisioning phase, privacy VM, host authority, or
  cross-repo sequencing is touched. The one lifecycle coupling: home-manager runs
  every activation entry in one `set -eu -o pipefail` script, so a non-zero exit
  from any entry aborts every entry the DAG sorts after it. The Pi installer and
  Claude merge are therefore invoked non-fatally (Interface Contracts): a refusal
  on a target collision or malformed operator `settings.json` prints a loud error
  and leaves the file untouched, but returns success, so it cannot skip any other
  activation entry (package install, generation link, `llmMemoryLinks`, systemd
  reload, hooks, branch-protection sync). Ordering is deliberately not relied on -
  the nixpkgs DAG `toposort` does
  not place this entry last (verified against the real allod-dev DAG:
  `installPackages` and `syncPrBranchProtection` sort after it), so containment
  comes from non-fatality, not from ordering.
- Persistent, mutable, operator-authoritative state is limited to
  `~/.claude/settings.json` plus the exact Pi extension target
  `~/.pi/agent/extensions/vm-status.ts`. The Claude change preserves settings
  (merge, not own) and guards them with the single-object preflight plus a
  failure-owning merge pipeline; the Pi change refuses to replace a non-managed
  file, directory, or symlink at the target path. Both refusal paths are exercised
  by behavioral fixture tests (below). This nudges the change above pure R1.
- Worst credible failure after planned validation passes: a broken generated
  `statusLine` script or Pi extension degrades or breaks harness startup on a dev
  VM. A non-zero/empty `statusLine` script only blanks Claude's status row (Claude
  tolerates it per its docs). A malformed Pi extension is the sharper risk - a
  parse/throw at load makes pi **refuse to start** - which is exactly what the Pi
  RPC render/sabotage tests cover (a broken extension exits non-zero with no
  `setStatus` event; a good one emits the event carrying the hostname). The
  settings.json risk is covered by the merge-script fixture tests (operator keys
  preserved and idempotent on valid input; every malformed shape refused with the
  file byte-unchanged). The Pi target-collision risk is covered by installer
  fixtures (missing path creates the managed symlink; an old managed store symlink
  is replaced; regular files, directories, and non-managed symlinks are refused
  byte-unchanged). The generated artifacts are deterministic templates with only
  the NixOS hostname interpolated where needed (neutralized via `builtins.toJSON`
  / `lib.escapeShellArg`), so the failure surface is small and fully exercised in
  fixtures/generated artifacts (principle 13). The R2 score rests on that fixture
  coverage plus the non-fatal mutable-state helper invocations, which keep a
  refusal from touching operator state or skipping any other activation entry;
  without the fixtures the honest score would drift to R3.
- Rollback is a straight revert plus a human rebuild; the only persistent runtime
  residue is the Pi extension symlink and the `.statusLine` key, both trivially
  removable, and both inert if left (see Rollback Plan).

Human scrutiny (look here first):

- The `settings.json` merge script's fixture tests: confirm they preserve every
  non-`statusLine` key, are idempotent, and refuse empty/scalar/multi-doc/malformed
  input with the file untouched.
- The Pi installer fixture tests: confirm they create only the managed symlink,
  replace only an old managed store symlink, and refuse a file/directory/custom
  symlink without touching it.
- The Pi RPC render test: confirm the real generated extension emits a `setStatus`
  event carrying `VM: <hostname>`, and that the sabotage variant fails closed.
- The Pi extension and Claude statusline script bake the same
  `osConfig.networking.hostName` value (one source, fanned out).

## Interface Contracts

**Module signature.** `modules/ai-agents.nix` inner module gains `osConfig`:
`{ identity, memoryCheckouts ? [...] }: { lib, pkgs, osConfig, ... }:` and reads
`hostName = osConfig.networking.hostName;` (for `allod-dev`, `"allod-dev"`). The
status text is `VM: ${hostName}`.

**Claude statusLine (verified against https://code.claude.com/docs/en/statusline).**

- settings shape (only `.statusLine` is module-managed; other keys preserved):

  ```json
  { "statusLine": { "type": "command", "command": "<abs path to script>" } }
  ```

- The `command` runs in a shell, receives the session JSON object on stdin, and
  its stdout (one row per line) renders above the built-in footer badges. Runs
  after each assistant message, after `/compact`, on permission/vim-mode change;
  debounced 300ms. Only runs once workspace trust is accepted (else
  `statusline skipped · restart to fix`).
- stdin JSON (relevant fields; there is **no** hostname field): `model.id`,
  `model.display_name`, `cwd`, `workspace.current_dir`, `workspace.project_dir`,
  `session_id`, `version`, `context_window.used_percentage`,
  `cost.total_cost_usd`, ... The script reads and discards stdin.
- Generated status script (hostname baked at generation):

  ```nix
  claudeStatusLine = pkgs.writeShellScript "claude-vm-statusline" ''
    cat >/dev/null 2>&1 || true
    printf '%s\n' ${lib.escapeShellArg "VM: ${hostName}"}
  '';
  ```

**Claude settings-merge script (factored out so it is directly testable).**

```nix
claudeStatusMerge = pkgs.writeShellScript "claude-vm-status-merge" ''
  set -eu
  settings="''${1:-$HOME/.claude/settings.json}"
  tmp="$settings.tmp"
  mkdir -p "$(dirname "$settings")"
  if [ -e "$settings" ]; then
    # Reject empty, scalar, array, multi-document, and malformed JSON in one gate.
    ${pkgs.jq}/bin/jq -e -s 'length==1 and (.[0]|type=="object")' "$settings" >/dev/null 2>&1 || {
      echo "ERROR: $settings is not a single JSON object; refusing to touch it. Fix or remove it, then rebuild." >&2
      exit 1
    }
    ${pkgs.jq}/bin/jq --arg cmd "${claudeStatusLine}" \
      '.statusLine = {type:"command",command:$cmd}' "$settings" > "$tmp" || {
        rm -f "$tmp"; echo "ERROR: failed to merge .statusLine into $settings" >&2; exit 1; }
  else
    ${pkgs.jq}/bin/jq -n --arg cmd "${claudeStatusLine}" \
      '{statusLine:{type:"command",command:$cmd}}' > "$tmp" || {
        rm -f "$tmp"; echo "ERROR: failed to write $settings" >&2; exit 1; }
  fi
  mv -f "$tmp" "$settings"
'';
```

**Pi extension (verified against installed `docs/extensions.md`, `docs/tui.md`,
`examples/extensions/status-line.ts`, and `dist/core/extensions/types.d.ts`).**

- Installed at `~/.pi/agent/extensions/vm-status.ts` (global auto-discovery); no
  `settings.json` entry required.
- `ctx.ui.setStatus(key, text)` sets a persistent footer status keyed by
  `"vm-status"`. `session_start` fires on `startup`/`new`/`resume`/`fork`/`reload`;
  footer state does not survive session replacement, so re-applying on
  `session_start` (as the shipped example does) is required for the indicator to
  persist across `/new`, `/resume`, `/fork`.
- Uses only `import type` (erased at runtime), so the symlinked store `.ts` loads
  under jiti with no `node_modules` resolution against the read-only store. The
  status text is passed to `setStatus` as a plain string - no `theme.fg` dim
  styling. Hostname-only is the MVP (Design Decision 3), and every extra UI call
  is runtime surface on the one artifact whose failure can stop `pi` from starting.

  ```nix
  piStatusExtension = pkgs.writeText "pi-vm-status.ts" ''
    import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
    export default function (pi: ExtensionAPI) {
      pi.on("session_start", async (_event, ctx) => {
        ctx.ui.setStatus("vm-status", ${builtins.toJSON "VM: ${hostName}"});
      });
    }
  '';
  ```

**Pi installer script (factored out so collision handling is directly testable).**

```nix
piStatusInstall = pkgs.writeShellScript "pi-vm-status-install" ''
  set -eu
  target="''${1:-$HOME/.pi/agent/extensions/vm-status.ts}"
  mkdir -p "$(dirname "$target")"

  if [ -e "$target" ] || [ -L "$target" ]; then
    if [ ! -L "$target" ]; then
      echo "ERROR: $target already exists and is not the managed VM status symlink; refusing to replace it." >&2
      exit 1
    fi
    current="$(readlink "$target")"
    case "$current" in
      /nix/store/*-pi-vm-status.ts) ;;
      *)
        echo "ERROR: $target points to $current, not the managed VM status extension; refusing to replace it." >&2
        exit 1
        ;;
    esac
  fi

  ln -sfnT ${piStatusExtension} "$target"
'';
```

**Activation contract (`home.activation.agentVmStatus`).** The Pi installer and
Claude merge are invoked NON-FATALLY; entry ordering is deliberately not relied
on. Home-manager runs every activation entry in one script under
`set -eu -o pipefail` (verified in the generated `activate`; the NixOS unit
launches it via `bash -el`), so any entry that exits non-zero aborts every entry
the DAG sorts *after* it. That ordering is not controllable. Home-manager sorts
the activation DAG with
`lib.hm.dag.topoSort cfg.activation` (`modules/lib/dag.nix` /
`modules/home-environment.nix`), which feeds `builtins.attrValues` (lexicographic
by entry name) into the nixpkgs `toposort`. A bare `entryAfter ["writeBoundary"]`
sorts `agentVmStatus` to the FRONT of the post-`writeBoundary` group, and even
anchoring it `entryAfter [ "writeBoundary" "llmMemoryLinks" "onFilesChange"
"reloadSystemd" "setupTrackedHooks" ]` does NOT make it last: evaluating the real
allod-dev activation DAG through that same `topoSort` still sorts `installPackages`
and `syncPrBranchProtection` after `agentVmStatus` (the DFS-based `toposort` does
not honor a name-anchor as "last"), so a fatal merge would still skip a package
install and the branch-protection sync. Name-anchoring is also fragile - any
future post-`writeBoundary` entry re-breaks it. So both mutable-state helpers are
invoked non-fatally and ordering is irrelevant:

```nix
home.activation.agentVmStatus = lib.hm.dag.entryAfter [ "writeBoundary" ] ''
  # Pi: install auto-discovered extension (immutable store source). Refusals on
  # operator-authored target collisions print a loud ERROR and leave the path
  # byte-unchanged, but must not abort the shared activation script.
  ${piStatusInstall} || true

  # Claude: merge ONLY .statusLine. A refusal on a malformed/empty operator
  # settings.json prints a loud ERROR to stderr (visible in the rebuild output and
  # `journalctl -u home-manager-<user>.service`) and leaves the file byte-unchanged.
  # It is invoked NON-FATALLY so that, under the shared `set -eu` activation script,
  # a refusal cannot abort activation or skip any entry the DAG sorts after this one.
  ${claudeStatusMerge} || true
'';
```

(The helper scripts still own their own preflight and atomicity; `|| true` only
stops the *shared* activation script from aborting - the loud ERROR and the
byte-unchanged file come from the helper itself. The flake check asserts both
non-fatal invocations - see Acceptance Tests.)

**Flake check.** Add `agent-vm-status` to `checks.<system>` mirroring the existing
`pi-integration` structure: embed `home.activation.agentVmStatus.data` via
`pkgs.writeText` (as `pi-integration` does). The activation text references
`piStatusInstall` and `claudeStatusMerge` directly; `piStatusExtension` is
referenced transitively from the installer and `claudeStatusLine` is referenced
transitively from the merge script. Nix string context on `.data` carries all
four into the `writeText` output's closure, so building the check realizes all
four into the store and makes the shell scripts executable in-sandbox - but
extract the Pi extension path from the installer script and the statusline path
from the merge script, not from the activation text (see Acceptance Tests). The
check's fixtures shell out to bare `jq`, so it must set
`nativeBuildInputs = [ pkgs.jq ]` (the merge script itself calls
`${pkgs.jq}/bin/jq` by absolute path and needs nothing on PATH; `cmp`, `mktemp`,
`grep`, `sed`, `readlink`, and `test` come from stdenv/coreutils). Then run the
artifact assertions and the installer/merge fixtures below. The Pi RPC
render/sabotage pair moves
into this check too if `pkgs-unstable.pi-coding-agent` runs under the build
sandbox with `HOME=$(mktemp -d)` and `--offline` (no TTY, no network); until that
is confirmed it stays a VM-side step (Acceptance Tests 4-5).

## Agent Gates

- **Live VM rebuild is human-gated.** The agent must not run `nixos-rebuild` or
  any host-side rebuild to make the change live. Blocks: the indicator only
  appears in running harnesses after a human rebuilds `allod-dev`, and the final
  live-TUI eyeball (below) can only happen post-rebuild. The agent **can** build
  and `nix eval` the generated artifacts and flake checks, run the Claude status
  and merge scripts against fixtures, and run the Pi RPC render/sabotage tests on
  `allod-dev` - none of which require a rebuild.
- No secrets, credentials, or provider API keys are involved; the tracking issue
  (`allod/profiles#7`) already exists. No other gates apply.

## Acceptance Tests

Run from `~/work/allod/profiles`. Assume `nix` with `nix-command flakes` enabled.

```sh
cd ~/work/allod/profiles
host="$(nix eval --raw .#nixosConfigurations.allod-dev.config.networking.hostName)"  # -> allod-dev
user="$(nix eval --raw .#vmFacts.allod-dev.username)"                                # -> allod

# --- 1. The new flake check must pass (this is the authoritative gate) ---
nix build --no-link .#checks.x86_64-linux.agent-vm-status

# The check (mirrored inline here) reads the activation script and extracts the
# two store artifacts it references DIRECTLY - the Pi installer and the Claude
# merge script. The Pi extension and Claude statusline scripts are NOT named in
# the activation text (their helper scripts invoke them), so extract those paths
# from the helper scripts. All four are realized in the check closure because
# the helpers reference them, and the `nix build` above already realized them.
act="$(nix eval --raw \
  ".#nixosConfigurations.allod-dev.config.home-manager.users.${user}.home.activation.agentVmStatus.data")"
printf '%s\n' "$act" | grep -F 'pi-vm-status-install'
printf '%s\n' "$act" | grep -F 'claude-vm-status-merge'
install="$(printf '%s\n' "$act" | grep -oE '/nix/store/[^" ]+-pi-vm-status-install'    | head -1)"
merge="$(printf '%s\n' "$act" | grep -oE '/nix/store/[^" ]+-claude-vm-status-merge'    | head -1)"
piext="$(grep -oE '/nix/store/[^" ]+-pi-vm-status\.ts' "$install"                     | head -1)"
claudesh="$(grep -oE '/nix/store/[^" ]+-claude-vm-statusline' "$merge"                 | head -1)"

# The mutable-state helpers MUST be invoked non-fatally: under the shared `set
# -eu` activation script a refusal must not abort activation and skip a later
# entry. In the real allod-dev DAG, installPackages and syncPrBranchProtection
# sort AFTER this entry, so a fatal refusal would skip a package install and the
# branch-protection sync.
printf '%s\n' "$act" | grep -E 'pi-vm-status-install[^|]*\|\| *(true|:)'
printf '%s\n' "$act" | grep -E 'claude-vm-status-merge[^|]*\|\| *(true|:)'

# Pi extension bakes the hostname and the required API calls:
grep -F "VM: ${host}" "$piext"
grep -F 'ctx.ui.setStatus("vm-status"' "$piext"
grep -F 'session_start' "$piext"

# --- 1a. Pi installer BEHAVIORAL fixtures (collision-safe symlink ownership) ---
work="$(mktemp -d)"
target="$work/home/.pi/agent/extensions/vm-status.ts"
"$install" "$target"
test -L "$target"
test "$(readlink "$target")" = "$piext"

# An old managed store symlink is replaceable, even if dangling after GC:
rm -f "$target"
ln -s /nix/store/old-pi-vm-status.ts "$target"
"$install" "$target"
test "$(readlink "$target")" = "$piext"

# Operator-authored regular files, directories, and custom symlinks are refused
# without being touched.
rm -f "$target"; printf 'operator extension' > "$target"; cp "$target" "$work/file.orig"
if "$install" "$target" 2>/dev/null; then echo "PI INSTALL REFUSAL FAILED for file" >&2; exit 1; fi
cmp "$target" "$work/file.orig" || { echo "PI INSTALL TOUCHED OPERATOR FILE" >&2; exit 1; }

rm -f "$target"; mkdir "$target"; before="$(find "$target" -maxdepth 1 -mindepth 1 | wc -l)"
if "$install" "$target" 2>/dev/null; then echo "PI INSTALL REFUSAL FAILED for directory" >&2; exit 1; fi
after="$(find "$target" -maxdepth 1 -mindepth 1 | wc -l)"
test "$before" = "$after" || { echo "PI INSTALL WROTE INSIDE OPERATOR DIRECTORY" >&2; exit 1; }

rmdir "$target"; ln -s "$work/custom.ts" "$target"; before="$(readlink "$target")"
if "$install" "$target" 2>/dev/null; then echo "PI INSTALL REFUSAL FAILED for custom symlink" >&2; exit 1; fi
test "$(readlink "$target")" = "$before" || { echo "PI INSTALL REPLACED CUSTOM SYMLINK" >&2; exit 1; }
rm -rf "$work"

# --- 1b. Claude merge script BEHAVIORAL fixtures (the real R2 coverage) ---
work="$(mktemp -d)"
# operator keys preserved + statusLine set + idempotent:
printf '{"model":"opus","theme":"dark"}' > "$work/s.json"
"$merge" "$work/s.json"
jq -e '.model=="opus" and .theme=="dark" and .statusLine.type=="command"' "$work/s.json" >/dev/null
# the merged command must point at the realized statusline script (one source, fanned out):
jq -e --arg c "$claudesh" '.statusLine.command==$c' "$work/s.json" >/dev/null
test -x "$claudesh"
cp "$work/s.json" "$work/s1.json"; "$merge" "$work/s.json"; cmp "$work/s1.json" "$work/s.json"  # idempotent (byte-identical)
# every malformed shape refused, file byte-unchanged (a real for-loop, not a
# pipe-to-while, so a failing case exits the script rather than a subshell):
bad=( '' '"scalar"' '[]' '{}
{}' '{ not json' )
for b in "${bad[@]}"; do
  printf '%s' "$b" > "$work/b.json"; cp "$work/b.json" "$work/b.orig"
  if "$merge" "$work/b.json" 2>/dev/null; then echo "REFUSAL FAILED for: $b" >&2; exit 1; fi
  cmp "$work/b.json" "$work/b.orig" || { echo "MERGE TOUCHED A REFUSED FILE: $b" >&2; exit 1; }
done
rm -rf "$work"

# --- 2. Claude RENDER contract (fully automated, the real contract): pipe a
#        representative stdin object and assert the hostname is in stdout ---
STDIN='{"model":{"display_name":"Opus","id":"claude-opus-4-8"},"workspace":{"current_dir":"/home/allod/work"},"cwd":"/home/allod/work","session_id":"test-abc","version":"2.1.204"}'
printf '%s' "$STDIN" | "$claudesh" | grep -F "VM: ${host}"

# --- 3. Claude NEGATIVE/sabotage: a wrong hostname must fail closed ---
sab="$(mktemp)"; sed "s/VM: ${host}/VM: WRONG-HOST/g" "$claudesh" > "$sab"; chmod +x "$sab"
if printf '%s' "$STDIN" | bash "$sab" | grep -qF "VM: ${host}"; then
  echo "SABOTAGE FAILED: mutated script still emitted real hostname" >&2; exit 1; fi
rm -f "$sab"

# --- 4. Pi RENDER proof (real runtime, headless, offline, no API key): pi --mode
#        rpc --offline loads the real extension, fires session_start, and emits the
#        setStatus UI event carrying the hostname. --offline guarantees no network
#        or provider-key dependency (matches the no-API-key claim). The "method"/
#        "statusKey" field names below are pinned to the installed pi's observed RPC
#        output; re-pin from a real run if the assertion fails. (Verified on
#        allod-dev: exit 0, event on stdout.) ---
ev="$(timeout 30 pi --mode rpc --offline -e "$piext" </dev/null 2>/dev/null)"
printf '%s\n' "$ev" | grep -F '"method":"setStatus"' | grep -F '"statusKey":"vm-status"' | grep -F "VM: ${host}"

# --- 5. Pi NEGATIVE/sabotage: a broken extension must fail closed - non-zero exit,
#        no setStatus event - proving the Pi load check can actually fail (#11).
#        (Verified: broken ext -> exit 1, "Failed to load extension: ParseError".) ---
sab="$(mktemp --suffix=.ts)"; printf 'export default function( {\n' > "$sab"
if timeout 30 pi --mode rpc --offline -e "$sab" </dev/null >"$sab.out" 2>/dev/null; then
  echo "SABOTAGE FAILED: broken extension loaded cleanly" >&2; exit 1; fi
grep -qF 'setStatus' "$sab.out" && { echo "SABOTAGE FAILED: setStatus emitted from broken ext" >&2; exit 1; }
rm -f "$sab" "$sab.out"

# --- 6. Full closure still evaluates ---
nix eval --raw .#nixosConfigurations.allod-dev.config.system.build.toplevel.drvPath >/dev/null
```

Not automatable (human, after rebuild): launch `pi` and `claude` on a live
`allod-dev` in a real terminal and confirm the footer/status row persistently
shows `VM: allod-dev` across turns and after `/new` (pi) / new messages (claude).
The literal footer glyph render needs a real PTY/TUI and, for Claude, an accepted
workspace-trust prompt (otherwise the row is suppressed - not a failure).

## Rollback Plan

Revert the `allod/profiles` commit/PR; a human rebuilds affected dev VMs. After
the revert lands and the store paths are garbage-collected, the generated
artifacts drop from the closure: Claude's status row silently blanks (docs-confirmed
tolerance) and the Pi extension symlink dangles - and pi **silently ignores** a
dangling discovered extension symlink (verified: no startup breakage), so the
residue is inert, not harmful. If an operator wants it gone immediately on a VM
that already activated:

```sh
rm -f ~/.pi/agent/extensions/vm-status.ts
tmp=$(mktemp) && jq 'del(.statusLine)' ~/.claude/settings.json > "$tmp" && mv "$tmp" ~/.claude/settings.json
```

(`sponge` is not installed on the VM; `jq | mv` is the portable form.) All other
settings keys are untouched by this change. No committed secret, credential, or
provider config is created, so there is nothing else to revoke.
