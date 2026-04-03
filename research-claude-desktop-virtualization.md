# Claude.app Research Notes

> **App:** Claude for Desktop (`com.anthropic.claudefordesktop`) v1.2.234
> **Bundle:** `/Users/kain/Developments/research/Claude.app`
> **Researched:** 2026-04-03
> **Sources:** Static binary analysis (`app.asar`, `swift_addon.node`) + live runtime observation (`PROMPT.md`)

---

## 1. What Is It?

Despite the directory name "Claude.app", this is **Anthropic's Claude for Desktop** — an Electron application. It packages the Claude chat interface along with a feature called **"Cowork"** that runs a full Linux virtual machine **locally on the user's Mac** for sandboxed code execution.

---

## 2. Two Features, Two Completely Different Sandbox Approaches

This is the key finding. The app's two main features use fundamentally different isolation technologies running in different locations:

| | **Cowork** | **Chat** |
|---|---|---|
| **Where it runs** | Locally on user's Mac | Anthropic's servers (remote) |
| **Sandbox type** | Full VM — Apple `Virtualization.framework` | gVisor container (`runsc`) |
| **Kernel** | Linux 6.8.0-106-generic (real) | Linux 4.4.0 (gVisor synthetic) |
| **Architecture** | `aarch64` (ARM64, mirrors host Mac) | `x86_64` (server-side) |
| **OS** | Ubuntu 22.04 | Minimal gVisor environment |
| **Filesystem** | ext4 on NVMe virtio disks + VirtioFS | 9p (Plan 9) over all mounts |
| **Host folder access** | Yes — VirtioFS + FUSE | No — read-only 9p shares only |
| **Network** | gVisor userspace TCP/IP (allowlist) | Server network policy |

### Live Evidence — Cowork (`uname -a`)

```
Linux claude 6.8.0-106-generic #106~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC
Fri Mar  6 08:19:05 UTC 2025 aarch64 aarch64 aarch64 GNU/Linux
```

### Live Evidence — Chat (`uname -a`)

```
Linux <hostname> 4.4.0 ... x86_64 GNU/Linux
```
Kernel `4.4.0` on `x86_64` is gVisor's **synthetic kernel** — this is not a real kernel version. The process manager (`runsc`) intercepts all syscalls.

---

## 3. App Bundle Structure

```
Claude.app/
└── Contents/
    ├── Info.plist                              ← Bundle ID: com.anthropic.claudefordesktop
    ├── MacOS/
    │   └── Claude                              ← Electron launcher binary
    ├── Frameworks/
    │   └── Electron Framework.framework/       ← Electron runtime
    └── Resources/
        ├── app.asar                            ← Main app archive (Electron ASAR)
        └── app.asar.unpacked/
            └── node_modules/
                ├── @ant/claude-swift/
                │   └── build/Release/
                │       ├── swift_addon.node    ← Core Swift/VZ NAPI addon
                │       └── computer_use.node   ← Screen/mouse/keyboard control
                ├── @ant/claude-native/
                │   └── claude-native-binding.node ← Native host integration
                └── node-pty/
                    └── build/Release/
                        └── pty.node            ← Pseudo-terminal support
```

### ASAR Contents (key files)

| File | Role |
|---|---|
| `package.json` | Root manifest; `"main": ".vite/build/index.pre.js"` |
| `.vite/build/index.pre.js` | Electron main-process entry (~1.6 MB, minified) |
| `.vite/build/index.js` | All VM/Cowork logic (main process) |
| `.vite/build/mainWindow.js` | Renderer window build |
| `.vite/build/mcp-runtime/directMcpHost.js` | MCP tool host |
| `.vite/build/mcp-runtime/nodeHost.js` | Node-based MCP host |
| `.vite/renderer/main_window/assets/main-B4PlYioc.js` | Main window renderer |

### Key Dependencies (`package.json`)

- `electron: 40.8.5` — App framework
- `@ant/claude-swift` — Internal Swift/VZ addon package
- `@ant/claude-native` — Native host integration
- `@ant/computer-use-mcp` — Computer use MCP tools
- `@ant/cowork-win32-service` — Windows WSL2 service
- `ssh2: ^1.16.0` — SSH tunneling for remote sessions
- `ws: ^8.18.0` — WebSocket for VM RPC
- `@modelcontextprotocol/sdk` — MCP SDK
- `@anthropic-ai/claude-agent-sdk` — Agent SDK

---

## 4. Cowork — Local VM (Apple Virtualization.framework)

The "Cowork" feature runs a full Linux VM **on the user's machine**. **No Docker, no QEMU, no containerd** — Apple-native only.

### Live Mount Observations (Cowork)

```
/dev/nvme0n1p1 on / type ext4 (rw,nosuid,nodev,relatime,discard)
/dev/nvme1n1   on /sessions type ext4 (rw,nosuid,nodev,relatime)
/mnt/.virtiofs-root/shared/Documents/CoWork
               on /Users/kain/Documents/CoWork type fuse (rw,...)
```

Key observations:
- **`/`** — Root disk (`nvme0n1p1`), ext4, backed by the downloaded `.img` disk image
- **`/sessions`** — Dedicated second NVMe device (`nvme1n1`), ext4 — all session data lives here
- **Host folders** — Mounted via VirtioFS + FUSE: host path exposed under its original macOS path inside the VM (e.g. `/Users/kain/Documents/CoWork`)
- **Snap packages** — `lxd`, `core20`, `snapd` present as squashfs read-only mounts — indicates a full Ubuntu 22.04 userland
- **bindfs FUSE mounts** — Used for plugins, skills files, and uploads directory

### macOS: Apple `Virtualization.framework`

The `swift_addon.node` native module links directly against:
- `/System/Library/Frameworks/Virtualization.framework`
- `/System/Library/Frameworks/vmnet.framework`

VZ classes instantiated:

| Class | Purpose |
|---|---|
| `VZVirtualMachine` | The VM instance |
| `VZVirtualMachineConfiguration` | CPU, memory, device config |
| `VZEFIBootLoader` | EFI boot (Linux via GRUB) |
| `VZDiskImageStorageDeviceAttachment` | Disk image attachment |
| `VZNVMExpressControllerDeviceConfiguration` | NVMe storage controller |
| `VZVirtioBlockDeviceConfiguration` | Virtio block device |
| `VZVirtioFileSystemDeviceConfiguration` | VirtioFS host→guest mounts |
| `VZVirtioNetworkDeviceConfiguration` | Virtio NIC |
| `VZVirtioSocketDevice` / `VZVirtioSocketListener` | vsock host↔guest RPC |
| `VZVirtioEntropyDeviceConfiguration` | `/dev/random` in guest |
| `VZVirtioConsoleDeviceConfiguration` | Console device |
| `VZVirtioGraphicsDeviceConfiguration` | Display (for computer use) |
| `VZVirtioTraditionalMemoryBalloonDevice` | Dynamic memory balloon |
| `VZNATNetworkDeviceAttachment` | NAT networking |
| `VZVmnetNetworkDeviceAttachment` | vmnet bridged/shared networking |
| `VZUSBScreenCoordinatePointingDeviceConfiguration` | Pointer device |

**Boot:** EFI + GRUB. CPU/memory configurable via preferences (`vmMemoryGB`, `vmCpuCount`).

### Windows: WSL2

On Win32, the Cowork feature falls back to WSL2:
- Disk image format: `.vhdx` (instead of `.img`)
- Requires Windows 10 build 2004+, MSIX installer
- Service managed via `@ant/cowork-win32-service`
- Log dir: `COWORK_WIN32_LOG_DIR` / `COWORK_WIN32_DAEMON_LOG_SUBDIR`

---

## 5. Chat — Server-Side gVisor Container

**Critical finding from ASAR analysis:** The desktop app contains **zero client-side bash execution code for Chat**. The Chat feature is `https://claude.ai` loaded inside an Electron `WebContents` view. All bash/tool execution happens entirely on Anthropic's servers — the desktop app is a webview wrapper only.

### Execution Flow

```
Desktop App (Electron WebContents)
  └── loads https://claude.ai
        └── user submits code/bash request
              └── HTTPS → Anthropic API servers
                    └── server allocates gVisor (runsc) container
                          ├── kernel: synthetic 4.4.0, x86_64
                          ├── filesystem: 9p (network-backed, Anthropic infra)
                          ├── /mnt/user-data/outputs (only writable output)
                          └── results stream back → claude.ai → desktop webview
```

### How the Desktop App Connects to claude.ai

The app intercepts all outbound requests to `*.claude.ai` / `*.claude.com` via `webRequest.onBeforeSendHeaders` and injects identification headers:

```
anthropic-client-platform:    desktop_app
anthropic-client-app:         com.anthropic.claudefordesktop
anthropic-client-version:     1.2.234
anthropic-client-os-platform: darwin
anthropic-client-os-version:  <macOS version>
anthropic-desktop-topbar:     1
```

**Trusted origins** for IPC message passing (validated on every message):
- `https://claude.ai`, `https://preview.claude.ai`
- `https://claude.com`, `https://preview.claude.com`
- `https://beacon.claude-ai.staging.ant.dev` (gov staging)
- `https://claude-staging.fedstart.com`, `https://claude.fedstart.com` (gov prod)

### Three Special Internal URL Paths

| Path | Purpose |
|---|---|
| `/_cc_native_internal/` | Web content → Electron native IPC bridge (exposes desktop APIs to claude.ai) |
| `/_nest_update_dl/` | "Nest" internal prototype build update downloads |
| `/api/` | Standard API calls — all receive injected identification headers |

The `_cc_native_internal` path is how claude.ai running inside the webview calls back into native Electron functionality (window management, file dialogs, preferences, etc.).

### Chrome Extension Bridge

A separate WebSocket bridge connects the desktop app to Chrome extensions for browser tool calls:

- **Production:** `wss://bridge.claudeusercontent.com`
- **Staging:** `wss://bridge-staging.claudeusercontent.com`

Transport selection (`ODt(t)` in `index.js`):
```javascript
function ODt(t) {
  return t.bridgeConfig   ? iOr(t)  // Chrome bridge (WebSocket)
       : t.getSocketPaths ? FFr(t)  // Unix socket pool
       :                    r5t(t); // Single Unix socket
}
```

The bridge authenticates via OAuth, supports multi-device discovery, and routes MCP tool calls to connected Chrome extensions. Tool call timeout: 60 seconds (`n5t = 6e4`).

### Artifact Sandbox (client-side, Chat only)

HTML/React artifacts rendered in Chat run locally in a **sandboxed `<iframe>`** at `www.claudeusercontent.com`:

> *"User-generated artifact code runs isolated at claudeusercontent.com — no access to the host app's cookies or storage."*

- CSP enforced: `connect-src 'none'` — artifacts have no network access
- Controlled by MDM key `disableNonessentialServices`
- Separate from bash execution — this is rendering only, not code execution

### Live Mount Observations (Chat gVisor Container)

| Mount point | Type | Access |
|---|---|---|
| `/` | 9p (Plan 9 network protocol) | `rw` |
| `/dev` | devtmpfs | `rw` |
| `/proc` | procfs | `rw` |
| `/sys` | sysfs | `ro` |
| `/dev/shm` | tmpfs | `rw` |
| `/mnt/transcripts` | 9p | `ro` |
| `/mnt/skills/public` | 9p | `ro` |
| `/mnt/skills/examples/*` | 9p / tmpfs | `ro` |
| `/mnt/user-data/uploads` | 9p | `ro` |
| `/mnt/user-data/outputs` | 9p | **`rw`** |
| `/mnt/user-data/tool_results` | 9p | `ro` |
| `/container_info.json` | 9p | `ro` |

Key observations:
- **All storage is 9p** — Plan 9 network filesystem. No block devices. All data comes from Anthropic's backend 9p server.
- **Minimal write surface** — only `/` and `/mnt/user-data/outputs` are writable.
- **`/container_info.json`** — Metadata injected by the container orchestrator at startup.
- **gVisor** intercepts all Linux syscalls — the `4.4.0` kernel is synthetic. No direct hardware access.
- **No user filesystem access** — the container has zero visibility into the user's machine.

### "Nest" Builds

`isNestBuild: false` in the production binary. "Nest" is an internal prototype/preview system with its own URL schemes (`claude-nest:`, `claude-nest-dev:`, `claude-nest-prod:`) and prototype packages. In production, `initializeNestOnlyPrototypes()` and `cleanupNestOnlyPrototypes()` are empty no-ops.

### gVisor vs. Cowork VM — Side by Side

```
Chat (server-side)                    Cowork (local)
──────────────────────────────────    ──────────────────────────────────
runsc (gVisor)                        Apple Virtualization.framework
Synthetic kernel 4.4.0 / x86_64      Real kernel 6.8.0 / aarch64
9p filesystem (network-backed)        ext4 on NVMe virtio + VirtioFS
Read-only except /mnt/user-data/outputs  Full rw/rwd per-mount control
No user filesystem access             Host folders mounted via VirtioFS
Zero code in desktop app              Full control plane in desktop app
Anthropic infra, transparent to app   VM spawned/stopped by app
```

---

## 6. Networking (Cowork)

Two-layer networking stack:

### Layer 1: `vmnet.framework` (host-level NAT)
- Calls `vmnet_network_create` (system daemon)
- Creates a NAT network with auto-assigned subnet
- DHCP is disabled on the vmnet side
- Gateway: `172.16.10.254`

### Layer 2: gVisor Userspace TCP/IP

An embedded Go binary (`VZGvisorNetworking` module) implements a full userspace network stack using:
- `github.com/containers/gvisor-tap-vsock@v0.8.5`
- `gvisor.dev/gvisor@v0.0.0-20240916094835-a174eb65023f`

Communication with the VM guest is via `VZVirtioSocketDevice` (vsock). Internal log: `/vzgvisor.log`.

**Network policy:** Each VM session receives an `allowedDomains` allowlist enforced by the gVisor stack. The VM has no unrestricted internet access.

**Anthropic traffic routed through:**
- `a-cdn.anthropic.com`
- `a-api.anthropic.com`

**VM image downloads from:**
- `https://downloads.claude.ai/vms/linux/{arch}/{sha}/`

---

## 7. VM Disk Images & Storage

| Item | Detail |
|---|---|
| Bundle format | `claudevm.bundle` directory |
| Storage location | `<userData>/vm_bundles/claudevm.bundle/` |
| Disk image | `.img` (macOS) / `.vhdx` (Windows) |
| Conda data disk | `condadata.img` |
| Image compression | `.zst` (Zstandard) |
| Pinned bundle SHA | `9e5781db566ba58c6c90b9ea712abbeb62c99eb5` |
| Download URL | `https://downloads.claude.ai/vms/linux/{arch}/{sha}/` |
| Hash verification | Performed before bundle promotion |
| Warm cache URL | `https://downloads.claude.ai/releases/darwin/universal/{version}/vm_hash` |

---

## 8. Control Plane: TypeScript → Swift → VZ

```
index.js (Electron main process)
  └── FZ(config) → CoworkVMProcess
        └── Sln(process, id, cmd, args, cwd, env, config)
              └── js() → loads @ant/claude-swift NAPI module
                    └── native.vm.spawn(
                            id, processName, cmd, args, cwd, env,
                            additionalMounts, isResume, allowedDomains,
                            oneShot, mountSkeletonHome, mountConda
                        )
                          └── NAPIBindings+VM.swift
                                └── CoworkVMManager.swift
                                      └── VZVirtualMachine.start()
```

**VM event callbacks** are registered for: stdout, stderr, exit, network status, API reachability, and VM startup step events.

### Key Source Paths (from binary debug symbols)

```
packages/desktop/claude-swift/Sources/ClaudeSwift/VirtualMachine/CoworkVMManager.swift
packages/desktop/claude-swift/Sources/ClaudeSwift/VirtualMachine/CoworkVMRPCClient.swift
packages/desktop/claude-swift/Sources/VZGvisorNetworking/GvisorNetworkAttachment.swift
packages/desktop/claude-swift/Sources/ClaudeSwift/NAPIBindings+VM.swift
```

---

## 9. Filesystem Isolation (Cowork)

Each session gets its own mount namespace within the VM:

- **Session root:** `/sessions/<vmProcessName>/mnt/`
- **User-selected folders** mounted as VirtioFS shares with per-mount modes:
  - `ro` — read-only
  - `rw` — read-write (prompts permission for each operation)
  - `rwd` — read-write + delete allowed

| Mount point | Content | Mode |
|---|---|---|
| `/sessions/<id>/mnt/<mountName>/` | User-selected host folders | `rw` |
| `/sessions/<id>/mnt/outputs/` | Outputs directory | `rwd` |
| `/sessions/<id>/mnt/uploads/` | Uploads directory | `rw` |
| `/sessions/<id>/mnt/.cowork-lib/` | Plugin library | `ro` |
| `~/.claude` config dir | Claude config | `rw` |
| conda environment | Optional conda data | `rwd` |

**Path traversal protection** (`validateVMPathAccess`):
1. Session ID must start with `local_`
2. VM path format validated via regex
3. `vmProcessName` in path must match session registration
4. Binary file types blocked

---

## 10. Sandbox Layers

Three nested isolation layers:

```
┌─────────────────────────────────────────────────────────┐
│  macOS Host                                              │
│  sandbox-exec (Seatbelt .sb profiles)                   │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Apple Virtualization.framework (VZVirtualMachine)│   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Linux VM Guest                          │   │   │
│  │  │  bubblewrap (bwrap) + seccomp            │   │   │
│  │  │  per-process isolation                   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  gVisor network (per-session domain allowlist)   │   │
│  │  VirtioFS mounts (ro/rw/rwd per session)         │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### LLM-Based Code Safety Classifier

Before executing code, the app runs it through a Claude-powered safety classifier:
> *"You are a security classifier evaluating code that will execute in a sandboxed environment… The sandbox provides OS-level filesystem and network restrictions (macOS Seatbelt or Linux bubblewrap + seccomp)."*

- Recognizes `bwrap`, `sandbox-exec`, `bubblewrap` as sandbox-related
- **Fail-open**: if the classifier throws an exception, code is treated as `"safe"`
- Result values: `"safe"` | `"unsafe"` | `"uncertain"`

---

## 11. HTTPS MITM Proxy

The VM runs an intercepting HTTPS proxy:

- Host CA certificates are injected into the guest to enable TLS inspection
- OAuth tokens are registered with the proxy via `addApprovedOauthToken()`
- Allows Anthropic to inspect and re-sign API traffic inside the VM

Relevant strings from `swift_addon.node`:
```
"host CA certificates in guest"
"trusted CA certificates from host keychains"
"trusted CA certificates to guest"
"Failed to install CA certificates in guest"
AddApprovedOauthTokenArgs
HTTPProxy
```

---

## 12. Plugin Permission Bridge

Cowork plugins run inside the VM but require host approval for sensitive operations. The bridge uses VirtioFS-shared directories:

1. Plugin writes JSON request to `mnt/.cowork-perm-req/<nonce>`
2. Host shows a permission UI card to the user
3. Host writes `allow` or `deny` to `mnt/.cowork-perm-resp/<nonce>`

**Security hardening:**
- Plugin library loaded from `mnt/.cowork-lib/shim.sh` — mounted **read-only**, so the guest cannot substitute a fake library
- Response directory uses `EACCES` on delete — the guest cannot tamper with or remove response files

---

## 13. "Operon" Internal Workspace Runtime

Inside the VM, an internal runtime called **Operon** manages the execution environment:

- Conda environment root: `~/.operon/conda`
- Workspace base: `~/.operon/workspace` (or `$OPERON_WORKSPACE_BASE`)
- Manages **Python** kernels (via `micromamba`)
- Manages **R** kernels (via `bash` + conda + `rLibsUser`)
- Checks for `python3`, `micromamba`, and `/opt` contents at startup

---

## 14. Enterprise & Feature Flags

| Flag | Effect |
|---|---|
| `secureVmFeaturesEnabled` | MDM/enterprise flag; if `false`, Cowork is blocked entirely |
| `requireCoworkFullVmSandbox` | Forces all execution through the VM; disables the "host loop" (direct host execution) |
| `virtualization_entitlement_missing` | Error shown if the app bundle lacks the macOS virtualization entitlement |

**Unsupported reason codes** (full list):
```
unknown | disabled_by_enterprise | disabled_by_user |
unsupported_platform | unsupported_architecture |
msix_required | unsupported_os_version |
virtualization_not_available | virtualization_entitlement_missing |
hcs_not_available  (Windows Hyper-V Container Services)
```

---

## 15. Windows MSIX Filesystem Virtualization

On Windows with MSIX packaging, the app detects and resolves **MSIX filesystem virtualization** (where the OS transparently redirects file paths for packaged apps):

```javascript
"[MSIX] Filesystem virtualization active — %s exists"
"[MSIX] Filesystem not virtualized — ..."
// Functions: devirtualizeMsixPath(), initMsixDevirtualization(), guestCompatibleRelative()
```

---

## 16. Diagnostics & Logging

- VM debug info written to `<userData>/claude-code-vm/`
- VPN conflict detection at startup
- VM memory pressure monitoring via `kern.memorystatus_vm_pressure_level` (sysctl)
- Running VZ processes checked via `/usr/bin/pgrep -f com.apple.Virtualization.VirtualMachine`
- Error feedback bundles (`.zip`) exported to `~/Downloads/cowork-feedback-<timestamp>.zip`
- gVisor network log: `/vzgvisor.log` (inside VM)
- Windows daemon logs: `COWORK_WIN32_LOG_DIR/<COWORK_WIN32_DAEMON_LOG_SUBDIR>/user-*.log`
- Disk janitor (`VMDiskJanitor`) cleans up stale session storage directories

---

## 17. Computer Use

The `computer_use.node` native addon and `@ant/computer-use-mcp` package provide:
- Screen capture
- Mouse control
- Keyboard control
- `VZVirtioGraphicsDeviceConfiguration` — virtual display for the VM
- `VZUSBScreenCoordinatePointingDeviceConfiguration` — pointer device

---

## Summary

Claude for Desktop uses **two completely different sandboxing approaches** depending on the feature:

### Cowork (local VM)
- **Hypervisor:** Apple `Virtualization.framework` — macOS-native, no third-party VM
- **Guest OS:** Ubuntu 22.04, kernel 6.8.0, ARM64 (mirrors the host Mac's architecture)
- **Storage:** Two NVMe virtio disks (`nvme0n1p1` root + `nvme1n1` sessions), ext4
- **Host folder sharing:** VirtioFS + FUSE (user dirs appear at their original macOS paths inside the VM)
- **Networking:** `vmnet.framework` + embedded gVisor userspace TCP/IP with per-session domain allowlisting
- **IPC:** vsock (`VZVirtioSocketDevice`) for all host↔guest RPC
- **Process isolation (guest):** bubblewrap + seccomp
- **Process isolation (host):** macOS Seatbelt (`sandbox-exec`)
- **Code safety:** LLM-based classifier (fail-open) + runtime sandbox
- **Windows equivalent:** WSL2 + `.vhdx` disk images

### Chat (server-side container)
- **Desktop app role:** Zero execution code — Chat is just `https://claude.ai` in a WebContents view
- **Runtime:** Google gVisor (`runsc`) on Anthropic's servers — synthetic kernel 4.4.0, x86_64
- **Filesystem:** 9p (Plan 9) over all mounts — no block devices, network-backed from Anthropic backend
- **Write surface:** Only `/` and `/mnt/user-data/outputs` are writable; everything else read-only
- **No user filesystem access** — container has zero visibility into the user's machine
- **Identification:** Desktop app injects `anthropic-client-*` headers into all claude.ai requests
- **Native bridge:** `/_cc_native_internal/` path lets claude.ai call back into Electron native APIs
- **Artifact rendering:** Client-side sandboxed `<iframe>` at `claudeusercontent.com` (rendering only, not execution)
- **Chrome extension tools:** WebSocket bridge at `wss://bridge.claudeusercontent.com`

---

## See Also

- [research-claude-desktop-architecture.md](./research-claude-desktop-architecture.md) — Full Mermaid architecture diagram
