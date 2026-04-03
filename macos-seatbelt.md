# macOS Seatbelt (Sandbox) — Technical Reference

> **Status:** `sandbox-exec` is officially deprecated by Apple, but remains functional and is used internally by macOS itself, Claude Code's `sandbox-runtime`, and many other security tools. The underlying kernel mechanisms are not going away.

---

## Table of Contents

1. [What is Seatbelt?](#what-is-seatbelt)
2. [Architecture Overview](#architecture-overview)
3. [Kernel Internals: MACF & Sandbox.kext](#kernel-internals-macf--sandboxkext)
4. [The Three Critical Hooks](#the-three-critical-hooks)
5. [SBPL — Sandbox Profile Language](#sbpl--sandbox-profile-language)
6. [sandbox-exec: The User-Space Entry Point](#sandbox-exec-the-user-space-entry-point)
7. [Enforcement Flow](#enforcement-flow)
8. [How Claude Code Uses Seatbelt](#how-claude-code-uses-seatbelt)
9. [Seatbelt vs Linux (bubblewrap + landlock + seccomp)](#seatbelt-vs-linux-bubblewrap--landlock--seccomp)
10. [Important Gotchas](#important-gotchas)
11. [Debugging Sandbox Violations](#debugging-sandbox-violations)
12. [Profile Reference](#profile-reference)

---

## What is Seatbelt?

Seatbelt (renamed "macOS Sandbox" in later OS versions) is Apple's **kernel-level mandatory access control** framework for processes. It predates containers entirely and works at the syscall boundary — every file open, network connect, process fork, or Mach port access is intercepted before the kernel honors it.

Key properties:

- **Deny-by-default:** Profiles start with `(deny default)` and selectively allow operations.
- **Irreversible:** Once `sandbox_init()` is called, a process cannot escape its profile — not even with a `root` exploit inside the sandbox.
- **Inherited:** All child processes spawned by a sandboxed process inherit the same restrictions.
- **OS-native:** No containers, no VMs, no extra daemons required (beyond `sandboxd`, which is always running).

---

## Architecture Overview

```mermaid
graph TD
    A["Claude Code / user process"] -->|"spawns bash via"| B["/usr/bin/sandbox-exec"]
    C["SBPL profile (.sb file)"] -->|"-f or -p flag"| B
    B -->|"calls sandbox_init() then exec()"| D["Sandboxed bash process"]

    D -->|"every syscall intercepted by"| E["MACF hooks (100+ intercepts)"]
    E --> F["Sandbox.kext — kernel extension"]
    F -->|"evaluates against profile"| G{Allow?}

    G -->|"yes"| H["XNU kernel honors syscall"]
    G -->|"no"| I["EPERM returned to process"]
    I -->|"violation logged via"| J["sandboxd daemon\n(com.apple.sandboxd / port 14)"]

    H --> K["Hardware: CPU, FS, network"]

    style A fill:#E1F5EE,stroke:#0F6E56,color:#085041
    style B fill:#FAECE7,stroke:#993C1D,color:#4A1B0C
    style C fill:#EAF3DE,stroke:#3B6D11,color:#173404
    style D fill:#FAECE7,stroke:#993C1D,color:#4A1B0C
    style E fill:#FAEEDA,stroke:#854F0B,color:#412402
    style F fill:#EEEDFE,stroke:#534AB7,color:#26215C
    style G fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
    style H fill:#E1F5EE,stroke:#0F6E56,color:#085041
    style I fill:#FCEBEB,stroke:#A32D2D,color:#501313
    style J fill:#FAEEDA,stroke:#854F0B,color:#412402
    style K fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
```

---

## Kernel Internals: MACF & Sandbox.kext

**MACF (Mandatory Access Control Framework)** is the XNU kernel's plugin system for security policy. Sandbox.kext registers itself as a MACF policy module at boot time.

```mermaid
flowchart LR
    subgraph XNU ["XNU Kernel"]
        direction TB
        BSD["BSD syscall layer"]
        MACF["MACF policy framework"]
        VFS["VFS / network stack"]
        BSD -->|"every operation"| MACF
        MACF -->|"if allowed"| VFS
    end

    subgraph KEXT ["Sandbox.kext"]
        direction TB
        Hooks["100+ hook functions\n(mpo_file_check_mmap,\nmpo_proc_check_fork, ...)"]
        Eval["sb_evaluate_internal()"]
        Profile["Compiled SBPL profile\n(per-process, from kauth cred)"]
        Hooks --> Eval
        Eval --> Profile
    end

    MACF -->|"calls registered hooks"| Hooks
    Profile -->|"allow / deny"| MACF

    style XNU fill:#E6F1FB,stroke:#185FA5
    style KEXT fill:#EEEDFE,stroke:#534AB7
```

Sandbox.kext is compiled into a KEXT (kernel extension) and communicates with the user-space `sandboxd` daemon via the special Mach port `HOST_SEATBELT_PORT` (port 14) and the XPC service `com.apple.sandboxd`.

---

## The Three Critical Hooks

Of the 100+ MACF hooks Sandbox.kext registers, three drive the core lifecycle:

```mermaid
sequenceDiagram
    participant P as Process
    participant K as XNU Kernel
    participant M as MACF
    participant S as Sandbox.kext

    Note over P,S: exec() called (e.g. sandbox-exec → bash)

    P->>K: execve(path, argv, envp)
    K->>M: mpo_proc_check_for()
    M->>S: Does this process need a sandbox profile?
    S-->>M: Yes — attach profile

    K->>M: mpo_cred_label_update_execve()
    Note over M,S: Binary fully loaded, not yet running
    M->>S: Create sandbox object, attach to kauth cred,<br/>remove Mach port access
    S-->>M: Profile attached — process is now sandboxed

    Note over P,S: Process begins executing

    P->>K: open("/etc/passwd", O_RDONLY)
    K->>M: mpo_file_check_open()
    M->>S: sb_evaluate_internal(cred, FILE_READ, "/etc/passwd")
    S-->>M: allow (matches profile rule)
    K-->>P: fd = 3

    P->>K: open("/Users/other/.ssh/id_rsa", O_RDONLY)
    K->>M: mpo_file_check_open()
    M->>S: sb_evaluate_internal(cred, FILE_READ, "~other/.ssh/id_rsa")
    S-->>M: deny
    K-->>P: EPERM
```

| Hook | When called | What it does |
|------|-------------|--------------|
| `mpo_proc_check_for` | At `exec()` start | Checks if this process needs a profile applied |
| `mpo_cred_label_update_execve` | Binary loaded, not yet running | Creates sandbox object, attaches to kauth credentials, removes Mach port access. This is the point of no return. |
| `sb_evaluate_internal` | Every intercepted operation | Looks up the process's compiled profile and evaluates the requested operation against it |

---

## SBPL — Sandbox Profile Language

SBPL is a **Scheme-like DSL** — a Lisp-syntax declarative policy language. Profiles are either pre-compiled named profiles or inline strings passed to `sandbox-exec`.

### Basic structure

```scheme
(version 1)
(deny default)              ; deny-all baseline — mandatory first line

; Allow reads of system libraries
(allow file-read*
    (subpath "/usr")
    (subpath "/System")
    (subpath "/Library"))

; Allow writes only in the working directory and /tmp
(allow file-write*
    (subpath (param "WORKING_DIR"))
    (subpath "/private/tmp")
    (literal "/dev/null"))

; Network: allow outbound only
(allow network-outbound)
(deny network-inbound)

; Process operations
(allow process-exec*)
(allow process-fork)
(allow sysctl-read)
```

### Path matchers

| Matcher | Matches |
|---------|---------|
| `(subpath "/foo")` | `/foo` and all descendants |
| `(literal "/foo/bar")` | Exactly `/foo/bar` only |
| `(regex "^/foo/.*\\.js$")` | Regex match |
| `(param "VAR")` | Value of `-D VAR=...` passed to `sandbox-exec` |
| `(string-append (param "HOME") "/.config")` | Concatenation |

### Operation categories

```mermaid
mindmap
  root((SBPL operations))
    file-read-*
      file-read-data
      file-read-metadata
      file-read-xattr
    file-write-*
      file-write-data
      file-write-create
      file-write-unlink
      file-write-mount
    network-*
      network-outbound
      network-inbound
      network-bind
      system-socket
    process-*
      process-exec
      process-fork
      process-signal
    mach-*
      mach-lookup
      mach-register
    ipc-*
      ipc-posix-sem
      ipc-posix-shm
    system-*
      sysctl-read
      sysctl-write
      system-privilege
```

### Important nuance: Unix sockets are classified as `network-outbound`

When a process calls `connect()` on a Unix domain socket (e.g. `/var/run/docker.sock`), Seatbelt classifies this as `network-outbound` — **not** a file operation. A `(deny file-write* ...)` rule on the socket path will not block the connection; you must use `(deny network-outbound)`.

---

## sandbox-exec: The User-Space Entry Point

`/usr/bin/sandbox-exec` is the CLI tool that applies a Seatbelt profile to a process:

```bash
# From a profile file
sandbox-exec -f profile.sb /path/to/command arg1 arg2

# Inline profile string
sandbox-exec -p '(version 1)(deny default)(allow file-read*)' /bin/bash

# With parameters (substituted into profile via (param "KEY"))
sandbox-exec -f profile.sb -D WORKING_DIR=/home/user/project /bin/bash
```

What happens internally:

```mermaid
flowchart TD
    A["sandbox-exec starts"] --> B["Parse -f / -p profile"]
    B --> C["Parse -D parameters"]
    C --> D["Call sandbox_init(profile, flags, &err)\nvia mac_syscall #381"]
    D --> E{Success?}
    E -->|"no"| F["Print error, exit 1"]
    E -->|"yes"| G["exec(command, args)\nProfile is now permanent"]
    G --> H["Command runs under sandbox restrictions\n(irreversible — cannot escape)"]

    style A fill:#EAF3DE,stroke:#3B6D11,color:#173404
    style D fill:#EEEDFE,stroke:#534AB7,color:#26215C
    style G fill:#FAECE7,stroke:#993C1D,color:#4A1B0C
    style H fill:#FAECE7,stroke:#993C1D,color:#4A1B0C
    style F fill:#FCEBEB,stroke:#A32D2D,color:#501313
```

The key: `sandbox_init()` calls the undocumented `mac_syscall(381, ...)` system call passing the string `"Sandbox"` as the module identifier. After this returns successfully, `exec()` is called — and the new process image inherits the sandbox profile permanently.

---

## Enforcement Flow

Every operation a sandboxed process attempts follows this evaluation path:

```mermaid
flowchart TD
    A["Sandboxed process attempts operation\n(open, connect, fork, exec, ...)"] --> B["Kernel intercepts via MACF hook"]
    B --> C["sb_evaluate_internal(kauth_cred, operation, object)"]
    C --> D["Look up compiled profile\nattached to process credentials"]
    D --> E["Walk profile rules in order"]
    E --> F{Rule matches?}
    F -->|"no more rules"| G["Apply default: (deny default)"]
    F -->|"match found"| H{Rule action}
    H -->|"allow"| I["Kernel proceeds with operation"]
    H -->|"deny"| J["Return EPERM / EACCES to process"]
    G --> J
    J --> K["sandboxd logs violation\nto syslog / Console.app"]

    style A fill:#E1F5EE,stroke:#0F6E56,color:#085041
    style C fill:#EEEDFE,stroke:#534AB7,color:#26215C
    style I fill:#E1F5EE,stroke:#0F6E56,color:#085041
    style J fill:#FCEBEB,stroke:#A32D2D,color:#501313
    style K fill:#FAEEDA,stroke:#854F0B,color:#412402
    style G fill:#FAEEDA,stroke:#854F0B,color:#412402
```

Rules are evaluated **first match wins**. Later rules do not override earlier ones (unlike some firewall models). Profiles are compiled from SBPL text into a binary format when `sandbox_init()` is called — evaluation is fast.

---

## How Claude Code Uses Seatbelt

Claude Code's `sandbox-runtime` generates an SBPL profile at launch time from your `settings.json` configuration and passes it inline to `sandbox-exec` via the `-p` flag:

```mermaid
sequenceDiagram
    participant U as User
    participant CC as Claude Code
    participant SR as sandbox-runtime
    participant SE as sandbox-exec
    participant BP as Sandboxed bash

    U->>CC: runs /sandbox command
    CC->>SR: read settings.json sandbox config
    SR->>SR: generate SBPL profile string<br/>(allowWrite paths, denyRead paths,<br/>allowed network domains)
    SR->>SE: sandbox-exec -p "generated SBPL" bash
    SE->>SE: sandbox_init() → exec(bash)
    SE->>BP: bash process starts (sandboxed)
    BP-->>CC: stdout/stderr piped back
    Note over BP: Cannot write outside CWD<br/>Cannot reach non-allowlisted domains<br/>Cannot read ~/.ssh, ~/.aws, etc.
```

The generated profile follows this structure:

```scheme
(version 1)
(deny default)

; Always allowed — system essentials
(allow file-read* (subpath "/usr"))
(allow file-read* (subpath "/System"))
(allow process-exec*)
(allow process-fork)
(allow sysctl-read)
(allow file-map-executable)  ; required for dyld

; Working directory — read + write
(allow file-read*  (subpath "/path/to/project"))
(allow file-write* (subpath "/path/to/project"))

; Temp dirs always writable
(allow file-write* (subpath "/tmp"))
(allow file-write* (subpath "/private/tmp"))
(allow file-write* (subpath "/private/var/folders"))

; Extra allowWrite paths from settings.json
(allow file-write* (subpath "/Users/me/.kube"))
(allow file-write* (subpath "/tmp/build"))

; Deny sensitive paths explicitly
(deny file-read* (subpath "/Users/me/.ssh"))
(deny file-read* (subpath "/Users/me/.aws"))

; Network — route through proxy
(allow network-outbound
    (remote unix-socket "/path/to/proxy.sock"))
(deny network-outbound)  ; block everything else
```

Network traffic from the sandbox is routed through a proxy running outside the sandbox that enforces the domain allowlist from `settings.json`.

---

## Seatbelt vs Linux (bubblewrap + landlock + seccomp)

Claude Code uses different mechanisms per OS to achieve the same two-boundary security model:

```mermaid
graph LR
    subgraph macOS ["macOS — Seatbelt"]
        SB["sandbox-exec\n+ SBPL profile"]
        SB --> SK["Sandbox.kext\n(MACF hooks)"]
        SK --> FS1["Filesystem isolation"]
        SK --> NET1["Network isolation"]
    end

    subgraph Linux ["Linux — Three-layer stack"]
        BW["bubblewrap\n(namespaces)"] --> FS2["Filesystem isolation\n(bind mounts, pivot_root)"]
        LL["landlock LSM"] --> FS3["Filesystem rules\n(path-level ACLs)"]
        SC["seccomp\n(BPF filters)"] --> SYS["Syscall filtering\n(blocks AF_UNIX socket creation)"]
    end
```

| Property | macOS Seatbelt | Linux bubblewrap+landlock+seccomp |
|----------|---------------|----------------------------------|
| Mechanism | Kernel extension (MACF hooks) | Namespaces + LSM + BPF |
| Filesystem isolation | SBPL `(allow/deny file-*)` | bubblewrap bind mounts + landlock rules |
| Network isolation | SBPL `(deny network-*)` + proxy | Network namespace (none) + SOCKS5 proxy |
| Syscall filtering | Implicit via MACF | Explicit seccomp BPF filter |
| Nested sandbox (Docker) | `enableWeakerNestedSandbox` mode | Requires `--privileged` or user namespaces |
| Documentation | Undocumented, deprecated API | Well-documented kernel features |
| Overhead | Minimal (kernel evaluation) | Minimal (BPF JIT compiled) |
| Granularity | Very fine (per-path, per-operation) | Fine (landlock) + coarse (namespaces) |
| Escape resistance | Very high (irreversible at exec) | High (namespace teardown requires CAP_SYS_ADMIN) |

---

## Important Gotchas

### 1. Deprecation — but not removal
`sandbox-exec` is deprecated. Apple wants developers to use the App Sandbox (entitlement-based, requires `.app` bundles). However, since Apple's own system daemons use Seatbelt profiles internally (see `/System/Library/Sandbox/Profiles/`), the underlying mechanism is unlikely to be removed anytime soon.

### 2. Unix sockets ≠ files
As noted above, `connect()` to a Unix socket is classified as `network-outbound` by Seatbelt. A `(deny file-write* (literal "/var/run/docker.sock"))` rule does **not** block Docker socket access. You must use:
```scheme
(deny network-outbound (remote unix-socket "/var/run/docker.sock"))
```

### 3. `file-map-executable` must be allowed
Without `(allow file-map-executable)`, `dyld` cannot map shared libraries into memory. This causes even trivial processes to crash. Always include it.

### 4. Computer use runs on the real desktop
When Claude Code uses computer use (controlling your screen on macOS), it runs **outside** the sandbox on your actual desktop. Seatbelt only applies to bash subprocesses.

### 5. Built-in file tools bypass the sandbox
Claude Code's `Read`, `Edit`, and `Write` file tools use the permission system directly — they do not run through the sandbox. Only bash commands and their children are sandboxed.

### 6. `enableWeakerNestedSandbox` mode
When running inside Docker on Linux, sandbox-runtime can't create full bubblewrap namespaces without `--privileged`. The `enableWeakerNestedSandbox` flag enables a weakened mode. Only use this when the Docker container itself provides the outer isolation boundary.

---

## Debugging Sandbox Violations

### Watch violations in real time

```bash
# Stream sandbox denials to your terminal
log stream --predicate 'process == "sandboxd"' --info

# Or filter kernel messages
log stream --predicate 'subsystem == "com.apple.sandbox"'
```

Violations look like:
```
sandboxd[1234]: bash(5678) deny file-read-data /Users/me/.ssh/id_rsa
sandboxd[1234]: bash(5678) deny network-outbound tcp 93.184.216.34:80
```

### Test a profile manually

```bash
# Run cat under a deny-all profile — should fail
sandbox-exec -p '(version 1)(deny default)' cat /etc/hosts
# → cat: /etc/hosts: Operation not permitted

# Verify working dir access
sandbox-exec -f profile.sb -D WORKING_DIR=/tmp \
    cat ~/.ssh/id_rsa
# → Operation not permitted  ✓ (blocked as expected)
```

### Iterative profile building

Start with `(deny default)` plus only `process-exec*` and `sysctl-read`, then run your target command and add `allow` rules for each denial logged by `sandboxd` until the application functions correctly.

---

## Profile Reference

### Minimal safe starter profile

```scheme
(version 1)
(deny default)

; Process execution
(allow process-exec*)
(allow process-fork)
(allow sysctl-read)

; Required for dyld (shared libraries)
(allow file-map-executable)

; File metadata (stat, readdir) — does not expose content
(allow file-read-metadata)

; System directories — read only
(allow file-read-data
    (subpath "/usr")
    (subpath "/bin")
    (subpath "/sbin")
    (subpath "/System")
    (subpath "/Library")
    (subpath "/private/etc")
    (subpath "/private/tmp")
    (subpath "/dev"))

; Project directory — full read/write
(allow file-read*  (subpath (param "WORKING_DIR")))
(allow file-write* (subpath (param "WORKING_DIR")))

; Temp always writable
(allow file-write*
    (subpath "/tmp")
    (subpath "/private/tmp")
    (subpath "/private/var/folders")
    (literal "/dev/null")
    (literal "/dev/tty"))

; Network — disable entirely
(deny network*)
```

### Enable outbound network via proxy

```scheme
; Replace (deny network*) with:
(allow network-outbound
    (remote unix-socket "/path/to/proxy.sock"))
(deny network*)
```

### Deny sensitive credential paths

```scheme
; Add after the working-dir allow rules:
(deny file-read* (subpath (string-append (param "HOME") "/.ssh")))
(deny file-read* (subpath (string-append (param "HOME") "/.aws")))
(deny file-read* (subpath (string-append (param "HOME") "/.gnupg")))
(deny file-read* (regex #".*\.env$"))
(deny file-read* (regex #".*secrets.*"))
```

---

*Last updated: April 2026. Based on Anthropic Claude Code sandbox-runtime source, Apple Developer documentation, and macOS security research.*
