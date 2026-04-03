# Claude Architecture Research

Static analysis and architecture research of Anthropic's Claude products — the desktop app and Chrome extension.

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

## Methodology

- **Static analysis:** `app.asar` extraction, binary string analysis of `.node` native addons
- **Runtime observation:** Live mount enumeration (`/proc/mounts`), `uname -a` inside each sandbox, network traffic inspection
- **Sources:** App bundle at `/Applications/Claude.app`, Chrome extension directory (v1.0.62)
