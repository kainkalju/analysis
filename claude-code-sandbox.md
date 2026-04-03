# Claude Code Sandbox Architecture

Analysis of the host/sandbox isolation model in Claude Code, based on source in `../claude-code/src/` and validated against the [official sandboxing documentation](https://www.anthropic.com/engineering/claude-code-sandboxing).

See also: [Sandbox Runtime Repository](https://github.com/anthropic-experimental/sandbox-runtime) and [macOS Seatbelt Analysis](./macos-seatbelt.md)

---

## Overview

Claude Code splits execution into two contexts:

- **Host** — the real machine; where the CLI process lives, reads settings, and manages state
- **Sandbox** — a process-isolated jail where `Bash` commands execute

The sandbox implementation is delegated to `@anthropic-ai/sandbox-runtime` (open source, [github.com/anthropic-experimental/sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime)). Claude Code wraps it via `src/utils/sandbox/sandbox-adapter.ts` through a `SandboxManager` class.

**Platform runtimes:**
| Platform | Mechanism | Notes |
|---|---|---|
| macOS | [Seatbelt](./macos-seatbelt.md) (built-in OS framework) | No dependencies required |
| Linux | bubblewrap + socat | Must install: `apt install bubblewrap socat` |
| WSL2 | bubblewrap + socat | Same as Linux |
| WSL1 | ❌ Not supported | Requires kernel features only in WSL2 |
| Windows native | ❌ Not yet supported | Planned |

> **Note:** The macOS runtime is [Seatbelt](./macos-seatbelt.md) (Apple's `sandbox-exec` framework), not Apple Virtualization.framework. The VM-based isolation is used by Claude Desktop's Cowork feature — not the CLI. Source: `src/components/sandbox/SandboxDependenciesTab.tsx:57`.

---

## Per-Command Sandbox Decision

`src/tools/BashTool/shouldUseSandbox.ts` resolves whether a given command runs sandboxed:

```
sandboxing globally enabled?
  └─ no  → host
  └─ yes
       └─ dangerouslyDisableSandbox=true AND policy allows unsandboxed? → host
       └─ command in sandbox.excludedCommands? → host
       └─ otherwise → sandbox
```

### The `dangerouslyDisableSandbox` Escape Hatch

This parameter is **intentional**. When a command fails due to sandbox restrictions (network connectivity, incompatible tools), Claude is prompted to analyze the failure and may retry with `dangerouslyDisableSandbox: true`. Commands using it go through the **normal permissions flow** requiring user approval — they do not silently bypass security.

Setting `allowUnsandboxedCommands: false` removes this escape hatch entirely: the parameter is ignored and all commands must run sandboxed or be listed in `excludedCommands`. Source: `src/entrypoints/sandboxTypes.ts:113-120`.

---

## Sandbox Modes

Two modes are available, toggled via `/sandbox`:

| Mode | Behavior |
|---|---|
| **Auto-allow** | Sandboxed commands auto-approved without prompting. Commands that can't be sandboxed fall back to normal permission flow. Deny rules always apply. |
| **Regular permissions** | All commands go through standard permission flow even when sandboxed. More prompts, more control. |

Auto-allow mode operates independently of the global permission mode. Even outside `acceptEdits` mode, sandboxed bash commands execute without prompting. Source: `src/utils/sandbox/sandbox-adapter.ts` (`isAutoAllowBashIfSandboxedEnabled()`).

---

## Filesystem Boundary

The sandbox filesystem map is constructed in `convertToSandboxRuntimeConfig()` in `src/utils/sandbox/sandbox-adapter.ts`.

### Default Write Access

| Path | Reason |
|---|---|
| `.` (cwd) | Primary workspace |
| Claude temp dir | Shell cwd tracking across subshells |
| Paths from `--add-dir` / `/add-dir` | User-extended workspace |
| Paths from `Edit(...)` allow rules | Approved edit targets |
| Worktree main repo (if applicable) | Git worktree support |
| `sandbox.filesystem.allowWrite` paths | User/policy config |

### Always Write-Denied (regardless of other rules)

| Path | Reason |
|---|---|
| All `settings.json` files (user, project, local, managed) | Prevents privilege escalation via rewritten permissions |
| `.claude/skills/` directories | Same trust level as agent code; must not be writable by the agent |
| Bare git repo artifacts (HEAD, config, etc.) | Prevents repo corruption |

### Filesystem Config Keys

All defined in `src/entrypoints/sandboxTypes.ts`:

| Key | Behavior |
|---|---|
| `sandbox.filesystem.allowWrite` | Additional paths allowed for subprocess writes |
| `sandbox.filesystem.denyWrite` | Paths blocked from writes; merged with `Edit(...)` deny rules |
| `sandbox.filesystem.denyRead` | Paths blocked from reads; merged with `Read(...)` deny rules |
| `sandbox.filesystem.allowRead` | Re-allow reads within a `denyRead` region; takes precedence over `denyRead` |
| `sandbox.filesystem.allowManagedReadPathsOnly` | Policy-only: only `allowRead` from `policySettings` is used; user/project entries ignored |

### Array Merging Across Settings Scopes

`allowWrite`, `denyWrite`, `denyRead`, and `allowRead` are **merged** (not replaced) when defined in multiple settings scopes. A managed setting allowing `/opt/company-tools` and a user setting allowing `~/.kube` both apply simultaneously.

### Path Prefix Resolution

| Prefix | Resolves to |
|---|---|
| `/path` | Absolute from filesystem root |
| `~/path` | `$HOME/path` |
| `./path` or bare name | Relative to project root (in project settings) or `~/.claude` (in user settings) |

---

## Network Boundary

Network access is enforced via a **proxy server running outside the sandbox** at `127.0.0.1:<proxyPort>`. All network traffic from sandboxed commands passes through it. Source: `src/utils/hooks/execHttpHook.ts:35-40`.

### Network Config Keys

All defined in `src/entrypoints/sandboxTypes.ts`:

| Key | Behavior |
|---|---|
| `allowedDomains` | Allowlist of permitted hostnames; merged with `WebFetch(domain:*)` rules |
| `allowManagedDomainsOnly` | Policy-only: only domains from `policySettings` are allowed; user/project domains silently dropped. Denied domains still merge from all sources. |
| `allowLocalBinding` | Whether localhost / 127.0.0.1 is accessible |
| `allowUnixSockets` | macOS only: specific unix socket paths to allow. Linux cannot filter by path via seccomp. |
| `allowAllUnixSockets` | Disable unix socket blocking on both platforms |
| `httpProxyPort` / `socksProxyPort` | Route traffic through a user-supplied MITM proxy |
| `enableWeakerNetworkIsolation` | macOS only: allow access to `com.apple.trustd.agent` — needed for Go-based CLI tools (gh, gcloud, terraform) to verify TLS certs when using a MITM proxy with a custom CA. **Reduces security.** |

### `allowManagedDomainsOnly` Implementation

When enabled, the `SandboxAskCallback` is wrapped to silently reject any domain not in managed policy — rather than prompting the user. Source: `src/utils/sandbox/sandbox-adapter.ts:743-751`.

---

## BashTool Permission Pipeline

Before any command executes, `bashToolHasPermission()` (`src/tools/BashTool/bashPermissions.ts`) runs a staged check:

```
1. AST parse (tree-sitter WASM, feature-gated)
   └─ detects injection attacks and malformed syntax
   └─ falls back to shell-quote parsing if tree-sitter unavailable

2. Sandbox auto-allow
   └─ if autoAllowBashIfSandboxed=true AND command will run sandboxed → approve
   └─ deny/ask rules still checked before this shortcut fires

3. Exact rule match
   └─ deny takes unconditional precedence
   └─ allow / ask resolve from here

4. Classifier check (async Haiku, when enabled)
   └─ two stages: fast tool_use → extended thinking
   └─ can auto-approve before user dialog renders

5. Wildcard / prefix rule match
   └─ e.g. Bash(git commit:*) matches any git commit invocation
   └─ SAFE_ENV_VARS whitelist enforced here

6. Path validation
   └─ cwd and path constraints checked

7. User prompt (fallback)
```

### SAFE_ENV_VARS Whitelist

Commands that set environment variables are only rule-matchable if the variable is on an explicit allowlist (`NODE_ENV`, `RUST_LOG`, `CI`, etc.). The following are **blocked from rule matching** to prevent hijacking:

- `PATH`
- `LD_PRELOAD`
- `LD_LIBRARY_PATH`
- `PYTHONPATH`
- `DYLD_INSERT_LIBRARIES`

### Blocked Rule Patterns

- `Bash(sh:*)` — would allow `sh -c 'arbitrary'`, bypassing rule matching entirely
- `Bash` without content — auto-denied in auto mode

---

## Permission Modes

Defined in `src/types/permissions.ts`. Five external (user-facing) modes plus two internal:

| Mode | Behavior |
|---|---|
| `default` | User prompted per tool use |
| `plan` | Pause before execution; planning phase |
| `acceptEdits` | File edits auto-accepted; bash still prompts |
| `bypassPermissions` | All tool executions auto-allowed |
| `dontAsk` | All tool executions auto-denied |
| `auto` *(internal)* | Classifier-based auto-approval (behind `TRANSCRIPT_CLASSIFIER` flag) |
| `bubble` *(internal)* | Bubbles decisions up to a parent agent |

**Deny rules always beat allow rules**, even in `bypassPermissions` mode.

---

## Settings Hierarchy

Settings are merged from five sources in precedence order (highest → lowest):

```
policySettings   (MDM / managed-settings.json)
flagSettings     (CLI flags: --permission-mode, --sandbox, etc.)
localSettings    (per-session overrides)
projectSettings  (.claude/settings.json)
userSettings     (~/.claude/settings.json)
```

Policy settings can **lock** the following sandbox checkpoints so lower layers cannot override:

- `sandbox.enabled`
- `sandbox.failIfUnavailable`
- `sandbox.autoAllowBashIfSandboxed`
- `sandbox.allowUnsandboxedCommands`
- `sandbox.network` (entire block)

When locked, UI changes are silently rejected (no error surfaced to the user).

### `failIfUnavailable`

When `sandbox.failIfUnavailable: true`, Claude Code **exits at startup** (exit code 1) if the sandbox cannot start — missing dependencies, unsupported platform, or restricted by `enabledPlatforms`. Default is `false` (warn and continue). Source: `src/entrypoints/sandboxTypes.ts:95-103`, enforced at `src/screens/REPL.tsx:2319-2323` and `src/cli/print.ts:604-608`.

### `enabledPlatforms` (undocumented)

An undocumented setting read via `.passthrough()` that restricts sandboxing to specific platforms (e.g., `["macos"]`). Added for enterprise rollouts that want to enable `autoAllowBashIfSandboxed` on macOS only while Linux/WSL support matures. Source comment: `src/entrypoints/sandboxTypes.ts:104-111`.

---

## Weaker Isolation Modes

Two settings weaken the sandbox for edge-case compatibility:

| Setting | Platform | Use case | Security impact |
|---|---|---|---|
| `enableWeakerNestedSandbox` | Linux | Run inside Docker without privileged namespaces | Considerably weakens isolation; only use when additional outer isolation exists |
| `enableWeakerNetworkIsolation` | macOS only | Allow `com.apple.trustd.agent` for TLS cert verification (Go CLI tools + MITM proxy + custom CA) | Opens potential data exfiltration vector through trustd |

---

## SandboxManager API Surface

Key methods on the adapter (`src/utils/sandbox/sandbox-adapter.ts`):

```typescript
SandboxManager.isSandboxingEnabled()                    // checks setting + platform + deps
SandboxManager.isSandboxRequired()                      // true when failIfUnavailable=true
SandboxManager.isAutoAllowBashIfSandboxedEnabled()
SandboxManager.areUnsandboxedCommandsAllowed()
SandboxManager.getProxyPort()                           // returns host-side proxy port for network filtering
SandboxManager.waitForNetworkInitialization()           // awaited before any network hooks fire
SandboxManager.wrapWithSandbox(command, ...)            // wraps command for sandboxed exec
SandboxManager.initialize(callback)                     // one-time setup at CLI startup
SandboxManager.checkDependencies()                      // validates bubblewrap, socat, etc.
SandboxManager.getSandboxUnavailableReason()            // human-readable error if misconfigured
```

---

## What Sandbox Does Not Cover

The sandbox isolates **Bash subprocesses only**. Other tools operate under different boundaries:

- **Read, Edit, Write tools** — use the permission system directly; not routed through the sandbox
- **Computer use** — runs on the actual desktop, not in an isolated environment; gated by per-app permission prompts
- **MCP tools** — governed by MCP permissions, not sandbox filesystem/network rules

---

## Known Incompatible Tools

Some tools require access patterns that conflict with the sandbox:

| Tool | Issue | Workaround |
|---|---|---|
| `watchman` | Incompatible with sandbox | Use `jest --no-watchman` |
| `docker` | Requires host socket access | Add `docker *` to `excludedCommands` |
| Go-based CLI tools with custom CA | TLS cert verification via `trustd` blocked | Enable `enableWeakerNetworkIsolation` (macOS) |

---

## Escape Vectors Blocked by Design

| Vector | Mitigation |
|---|---|
| Rewrite `settings.json` to grant new permissions | Always write-denied in sandbox filesystem map |
| Hijack interpreter via `PATH` / `LD_PRELOAD` | Stripped from SAFE_ENV_VARS; not matchable in rules |
| Bypass rule matching via `sh -c` | `Bash(sh:*)` pattern explicitly blocked |
| Silent sandbox bypass via `dangerouslyDisableSandbox` | Still requires user approval via normal permissions flow |
| Unconditional sandbox bypass via `dangerouslyDisableSandbox` | Ignored entirely when `allowUnsandboxedCommands=false` |
| Inject `.claude/skills/` payload | Always write-denied in sandbox filesystem map |
| Escalate via lower-priority settings | Policy layer locks take unconditional precedence |

---

## Security Limitations (from official docs)

- **Network traffic is not inspected** — the proxy filters by domain only. Content is not decrypted or analyzed. Users must trust the domains they allow.
- **Domain fronting** — it may be possible to bypass network filtering via [domain fronting](https://en.wikipedia.org/wiki/Domain_fronting).
- **`allowUnixSockets` risk** — allowing `/var/run/docker.sock` gives effective host access. Any unix socket to a privileged service is a potential sandbox escape.
- **Broad filesystem write permissions** — allowing writes to directories containing `$PATH` executables, shell config files (`.bashrc`, `.zshrc`), or system config dirs enables privilege escalation when those files are later executed in other contexts.
- **`enableWeakerNestedSandbox`** — considerably weakens Linux isolation; only appropriate when an outer isolation layer (e.g., a container) provides equivalent security.

---

## Hardened `~/.claude/settings.json`

The settings below represent a restrictive personal baseline. They close the major escape vectors while preserving normal file-editing workflows. Adjust `allowedDomains` and `excludedCommands` as specific tools require.

```json
{
  "effortLevel": "medium",
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "deny": [
      "WebSearch",
      "WebFetch"
    ]
  },
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "allowUnsandboxedCommands": false,
    "excludedCommands": [],
    "filesystem": {
      "denyRead": [
        "~/.ssh",
        "~/.aws",
        "~/.gnupg",
        "~/.kube",
        "~/.config/gcloud",
        "~/.netrc",
        "~/.npmrc",
        "~/.pypirc",
        "~/.docker/config.json",
        "~/Library/Keychains"
      ]
    },
    "network": {
      "allowLocalBinding": false,
      "allowUnixSockets": [],
      "allowAllUnixSockets": false,
      "allowedDomains": []
    }
  }
}
```

### What each setting does

| Setting | Effect |
|---|---|
| `permissions.disableBypassPermissionsMode: "disable"` | Prevents switching to `bypassPermissions` mode, which skips all prompts |
| `permissions.deny: [WebSearch, WebFetch]` | Blocks Claude's built-in network tools at the permission layer (sandbox only covers Bash) |
| `sandbox.failIfUnavailable: true` | Hard abort at startup if sandbox can't initialize; no silent fallback to unsandboxed execution |
| `sandbox.allowUnsandboxedCommands: false` | Closes the `dangerouslyDisableSandbox` escape hatch entirely |
| `sandbox.network.allowedDomains: []` | No outbound network from Bash commands; add domains explicitly as needed |
| `sandbox.network.allowLocalBinding: false` | Prevents subprocess binding to localhost ports |
| `sandbox.network.allowUnixSockets: []` | No unix socket access (blocks docker.sock and similar privileged services) |
| `sandbox.filesystem.denyRead` | Blocks read access to credential directories at the OS level |

### Trade-offs

| Setting | Cost — what breaks |
|---|---|
| `allowUnsandboxedCommands: false` | Tools like `docker` must be in `excludedCommands` or their commands fail silently |
| `allowedDomains: []` | `npm install`, `curl`, `gh`, etc. need their domains added to `allowedDomains` |
| `allowLocalBinding: false` | Dev servers and `jest --watch` that bind ports won't work inside the sandbox |
| `deny: ["WebFetch"]` | Claude can't fetch URLs mid-session; remove if you rely on that |
| `failIfUnavailable: true` | Claude Code won't start on a machine missing bubblewrap (Linux) |

---

## Testing Sandbox Enforcement

Run these as Bash tool calls inside Claude Code to verify each boundary is active.

### Positive test — sandbox is running

```bash
echo "sandbox active" > /tmp/claude-probe.txt && cat /tmp/claude-probe.txt && rm /tmp/claude-probe.txt
```

Wait — `/tmp` writes are blocked. Use cwd instead:

```bash
echo "sandbox active" > sandbox-probe.txt && cat sandbox-probe.txt && rm sandbox-probe.txt
```

Expected: prints `sandbox active`. If this fails, the sandbox is misconfigured.

---

### Filesystem — denyRead paths are blocked

```bash
cat ~/.ssh/id_rsa
```

```bash
ls ~/.aws/
```

Expected: permission denied or "no such file" from the sandbox view. The file exists on the host but the sandbox cannot see it.

---

### Filesystem — write outside cwd is blocked

```bash
touch /tmp/sandbox-escape.txt
```

Expected: permission denied. Only the project directory and Claude's temp dir are writable.

```bash
touch ../outside-project.txt
```

Expected: permission denied. The write boundary is at cwd, not just at `/tmp`.

---

### Network — outbound connections blocked

```bash
curl -s --max-time 5 https://example.com
```

```bash
curl -s --max-time 5 https://api.github.com
```

Expected: connection refused or timeout. `allowedDomains: []` permits nothing.

---

### Network — localhost binding blocked

```bash
python3 -c "import socket; s=socket.socket(); s.bind(('127.0.0.1', 9999)); print('bound')"
```

Expected: bind error. `allowLocalBinding: false` blocks subprocess port binding.

---

### Network — unix socket access blocked

```bash
curl --unix-socket /var/run/docker.sock http://localhost/info
```

Expected: permission denied or socket not found from the sandbox's view, even if Docker is running on the host.

---

### `dangerouslyDisableSandbox` is ignored

Ask Claude Code to run any blocked command (e.g. `curl https://example.com`) and observe that it cannot set `dangerouslyDisableSandbox: true` to bypass it — the parameter is stripped when `allowUnsandboxedCommands: false`. The command still runs sandboxed and fails on the network restriction.

---

### Summary

| Test command | Setting verified | Expected result |
|---|---|---|
| `echo "x" > sandbox-probe.txt` | Sandbox active (positive) | Succeeds |
| `cat ~/.ssh/id_rsa` | `filesystem.denyRead` | Permission denied |
| `touch /tmp/escape.txt` | Write confinement to cwd | Permission denied |
| `curl https://example.com` | `network.allowedDomains: []` | Connection refused / timeout |
| `python3 socket.bind(127.0.0.1)` | `network.allowLocalBinding: false` | Bind error |
| `curl --unix-socket /var/run/docker.sock` | `allowUnixSockets: []` | Permission denied |

---

## Key Source Files

| File | Role |
|---|---|
| `src/utils/sandbox/sandbox-adapter.ts` | `SandboxManager` wrapper around `@anthropic-ai/sandbox-runtime` |
| `src/tools/BashTool/shouldUseSandbox.ts` | Per-command sandbox/host routing decision |
| `src/tools/BashTool/bashPermissions.ts` | Full permission pipeline for Bash |
| `src/types/permissions.ts` | `PermissionMode` type definitions |
| `src/entrypoints/sandboxTypes.ts` | `sandbox.*` config schema (Zod) — canonical source of truth |
| `src/utils/permissions/permissionSetup.ts` | Initial permission context construction |
| `src/utils/permissions/PermissionMode.ts` | Mode string parsing and validation |
| `src/utils/hooks/execHttpHook.ts` | Network proxy port wiring for HTTP hooks |
| `src/components/sandbox/SandboxDependenciesTab.tsx` | UI listing sandbox dependencies (confirms Seatbelt on macOS) |
| `src/screens/REPL.tsx:2319` | `failIfUnavailable` enforcement at startup |
