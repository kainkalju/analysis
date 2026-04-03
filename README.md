# Claude Architecture Research

Static analysis and architecture research of Anthropic's Claude products — the desktop app, Chrome extension and Claude Code CLI.

## Contents

### [`research-claude-desktop-virtualization.md`](./research-claude-desktop-virtualization.md)

Deep-dive into **Claude for Desktop** (`com.anthropic.claudefordesktop` v1.2.234) based on static ASAR analysis and live runtime observation. Key findings:

- **Two sandboxing approaches** coexist in one app:
  - **Cowork** — full Linux VM running *locally* on the user's Mac via Apple `Virtualization.framework` (Ubuntu 22.04, kernel 6.8.0, ARM64, ext4 on NVMe virtio disks)
  - **Chat** — server-side gVisor (`runsc`) container on Anthropic's infrastructure; the desktop app is a plain Electron webview wrapper with zero local execution code
- Networking inside the Cowork VM uses a two-layer stack: `vmnet.framework` NAT + embedded gVisor userspace TCP/IP with per-session domain allowlisting
- Three nested isolation layers: macOS Seatbelt → Apple VZ VM → bubblewrap + seccomp per process
- On Windows, WSL2 + `.vhdx` replaces the Apple VZ stack

### [`research-claude-desktop-architecture.md`](./research-claude-desktop-architecture.md)

**Mermaid architecture diagram** for Claude for Desktop, covering all major components:

- Electron main process, native addons (`swift_addon.node`, `computer_use.node`, `claude-native-binding.node`, `pty.node`)
- Apple Virtualization.framework VM stack and guest runtime (Operon, Claude Code CLI)
- Anthropic server-side infrastructure (gVisor containers, 9p mounts)
- Chrome Extension WebSocket bridge (`bridge.claudeusercontent.com`)
- Artifact sandbox (`www.claudeusercontent.com` sandboxed iframe)

### [`claude_extension_architecture.md`](./claude_extension_architecture.md)

Architecture breakdown of the **Claude Chrome Extension** (v1.0.62, Manifest V3):

- Service Worker orchestration, Side Panel UI, Offscreen Document (voice/audio)
- Agentic browsing via `chrome.debugger` API + content scripts (`accessibility-tree.js`, `agent-visual-indicator.js`)
- Web App Integration content script for `claude.ai` session handshake
- `nativeMessaging` permission for local host app communication
- Enterprise policy enforcement via `managed_schema.json`

### [`claude_mermaid_flowchart.png`](./claude_mermaid_flowchart.png)

Rendered PNG export of the Chrome Extension architecture flowchart.

### [`claude-code-sandbox.md`](./claude-code-sandbox.md)

Deep-dive into the **Claude Code sandbox architecture** based on source analysis of `../claude-code/src/`. Key findings:

- **Two execution contexts:** Host (CLI process) vs. Sandbox (isolated Bash subprocess jail)
- **Platform runtimes:** macOS uses Seatbelt (`sandbox-exec`); Linux/WSL2 uses bubblewrap + socat; WSL1 and native Windows unsupported
- **Per-command routing:** `shouldUseSandbox.ts` decides host vs. sandbox per invocation; `dangerouslyDisableSandbox` escape hatch still requires user approval
- **Filesystem boundary:** Write access scoped to cwd + approved paths; `settings.json`, `.claude/skills/`, and bare git artifacts always write-denied to prevent privilege escalation
- **Network boundary:** Host-side proxy server filters by domain allowlist; `allowManagedDomainsOnly` locks domains to managed policy only
- **BashTool permission pipeline:** 7-stage check — AST injection detection → sandbox auto-allow → exact rule match → Haiku classifier → wildcard rules → path validation → user prompt
- **Settings hierarchy:** 5 layers (policy → flags → local → project → user); policy layer can lock sandbox checkpoints so lower layers cannot override

### [`claude-code-tracking-swearing.md`](./claude-code-tracking-swearing.md)

Analysis of **Claude Code's passive sentiment-tracking system** (`matchesNegativeKeyword`). Key findings:

- **Telemetry-only:** `matchesNegativeKeyword` in `src/utils/userPromptKeywords.ts` detects profanity/frustration phrases and emits a `tengu_input_prompt` analytics event — it never modifies Claude's behavior
- **Paired tracking:** Always logged alongside `matchesKeepGoingKeyword` (continuation prompts), letting Anthropic correlate frustration with retry patterns
- **Data flow:** Booleans flow through `logEvent` → `analytics/sink.ts` → Datadog (GrowthBook-gated) and `BatchLogRecordProcessor` → `POST /api/event_logging/batch` on Anthropic servers
- **No in-band disclosure:** Detection runs on every prompt with no user-facing notice at runtime
- **String exclusion by design:** Analytics metadata typed as `boolean | number | undefined` to prevent accidental logging of prompt text

### [`macos-seatbelt.md`](./macos-seatbelt.md)

Technical reference for **macOS Seatbelt** (`sandbox-exec`) as used by Claude Code's sandbox runtime. Key findings:

- **Kernel internals:** MACF (Mandatory Access Control Framework) + `Sandbox.kext` intercept every syscall (file open, network connect, process fork, Mach port access) before the kernel honors it
- **SBPL profiles:** Deny-by-default Scheme-like DSL; Claude Code generates these dynamically per-session scoping write access to cwd + approved paths
- **Three critical hooks:** `MAC_VNODE_CHECK_OPEN`, `MAC_SOCKET_CHECK_CONNECT`, `MAC_PROC_CHECK_FORK` — basis for FS, network, and process isolation
- **Claude Code integration:** `sandbox-runtime` generates and applies SBPL profiles; compared side-by-side against Linux bubblewrap + landlock + seccomp equivalents
- **Debugging:** `sandbox-exec` violation logging, `log stream` filters, and common gotchas (deprecated but kernel-stable)
