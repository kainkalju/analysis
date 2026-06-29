# Manus Desktop App - Technical Architecture Overview

**Version:** 1.6.2  
**Platform:** macOS (arm64, Apple Silicon optimized)  
**Bundle ID:** `im.manus.desktop`  
**Build System:** Electron + Node.js + TypeScript  
**Minimum macOS Version:** 11.0  
**Frontend:** Next.js + React  

---

## Executive Summary

Manus is a cross-platform desktop application built with Electron that provides a task management and productivity interface. The app follows a modern multi-process Electron architecture with:
- **Main Process**: Handles system lifecycle, IPC, and file I/O
- **Renderer Process**: Next.js frontend with React UI
- **Sidecar Process**: Backend service orchestration via WebSocket
- **Helper Processes**: GPU, Renderer, and Plugin delegated processes

The app communicates with a cloud backend via WebSocket (Socket.io) using device and session authentication. It manages local development environments ("MyComputer") through spawned sidecar processes.

---

## Architecture Overview

```mermaid
graph TB
    subgraph "macOS System"
        direction TB
        ELF["Manus Main Executable<br/>(arm64 Mach-O)"]
        EFRAME["Electron Framework<br/>(154MB - native bridge)"]
        HELPERS["Helper Processes<br/>- GPU Worker<br/>- Renderer Worker<br/>- Plugin Worker"]
    end
    
    subgraph "Electron Main Process"
        MP["Main Process<br/>(index.js)"]
        WM["Window Manager"]
        IPC["IPC Router"]
        PROTOCOL["Custom Protocol Handler<br/>(app:// manus://)"]
        STORAGE["Storage Helper<br/>(localStorage, keychain)"]
        AUTO["Auto Updater<br/>(electron-updater)"]
    end
    
    subgraph "MyComputer System"
        MC["MyComputer<br/>(Device Manager)"]
        SIDECAR["Sidecar Manager<br/>(Spawns child processes)"]
        SC["Sidecar Processes<br/>(Backend services)"]
    end
    
    subgraph "Network & Backend"
        WS["Socket.io Connection<br/>(WebSocket)"]
        BACKEND["Manus Cloud Backend<br/>(Device Config, Auth)"]
    end
    
    subgraph "Renderer Process"
        PRELOAD["Preload Script<br/>(Context Bridge)"]
        FRONTEND["Next.js Frontend<br/>(React UI)"]
        WINDOWS["- Main Window<br/>- Quick Ask Window<br/>- Presenter Window<br/>- Child Windows"]
    end
    
    ELF --> EFRAME
    EFRAME --> MP
    MP --> WM
    MP --> IPC
    MP --> PROTOCOL
    MP --> STORAGE
    MP --> AUTO
    MP --> MC
    
    MC --> SIDECAR
    SIDECAR --> SC
    MC --> WS
    WS --> BACKEND
    
    MP --> PRELOAD
    PRELOAD --> FRONTEND
    FRONTEND --> WINDOWS
    IPC --> WINDOWS
    PROTOCOL --> FRONTEND
    
    EFRAME --> HELPERS
```

---

## Process Model & Communication

```mermaid
graph TB
    subgraph "Primary Process Tree"
        MAIN["Manus Main Process<br/>(PID: parent)"]
        RENDERER["Renderer Process<br/>(child)"]
        SIDECAR1["Sidecar-1<br/>(child)"]
        SIDECAR2["Sidecar-2<br/>(child)"]
    end
    
    subgraph "IPC Channels"
        CH1["IPC Channel 1<br/>(Main ↔ Renderer)"]
        CH2["WebSocket<br/>(Main ↔ Backend)"]
        CH3["Stdio/Pipe<br/>(Main → Sidecars)"]
    end
    
    subgraph "Communication Protocols"
        ELECTRON["Electron IPC<br/>- ipcMain.on<br/>- ipcRenderer.send"]
        SOCKET["Socket.io<br/>- auth: token+deviceId<br/>- transports: websocket"]
        STDIO["Child Process<br/>- stdio streams<br/>- exit/error signals"]
    end
    
    MAIN --> CH1
    MAIN --> CH2
    MAIN --> CH3
    
    CH1 --> RENDERER
    CH2 --> BACKEND["Backend"]
    CH3 --> SIDECAR1
    CH3 --> SIDECAR2
    
    CH1 --> ELECTRON
    CH2 --> SOCKET
    CH3 --> STDIO
    
    style MAIN fill:#4A90E2
    style RENDERER fill:#7ED321
    style SIDECAR1 fill:#F5A623
    style SIDECAR2 fill:#F5A623
```

---

## Main Process Lifecycle

```mermaid
sequenceDiagram
    participant App as App Launch
    participant Main as Main Process
    participant Storage as Storage
    participant Backend as Cloud Backend
    participant Window as Window Manager
    participant Frontend as Frontend UI

    App->>Main: app.whenReady()
    Main->>Main: initLogs()
    Main->>Main: setupCorsBypass()
    Main->>Main: registerAppProtocol('app://', 'manus://')
    
    Main->>Storage: Load session token
    alt Token exists
        Main->>Backend: Connect WebSocket
        Backend-->>Main: device_config received
    else No token
        Main->>Window: Create login window
    end
    
    Main->>Window: Create main window
    Window->>Window: Create BrowserWindow
    Window->>Frontend: Load Next.js app (app://manus/app)
    Frontend->>Frontend: Render UI
    
    Main->>Main: Setup IPC handlers
    Main->>Main: Setup shortcuts
    Main->>Main: Setup menu bar
    Main->>Main: Setup tray
    
    Frontend-->>Main: Request myComputer state
    Main-->>Frontend: Send device sessions
```

---

## Module Architecture

### 1. Window Management System

```mermaid
graph TB
    subgraph "Window Manager"
        WM["WindowManager<br/>(Singleton)"]
        MW["MainWindow<br/>(type: main)"]
        QW["QuickAskWindow<br/>(type: quickAsk)"]
        CW["Child Windows<br/>(transient)"]
        PW["Presenter Window<br/>(full-screen mode)"]
    end
    
    subgraph "Window Features"
        STATE["Window State Manager<br/>(electron-window-state)"]
        BOUNDS["Bounds Tracking<br/>(position, size)"]
        PERSIST["Persistence<br/>Save/restore state"]
    end
    
    subgraph "IPC Events"
        EV1["open-child-window"]
        EV2["open-presenter-window"]
        EV3["presenter-ready"]
        EV4["window-minimize/maximize/close"]
    end
    
    WM --> MW
    WM --> QW
    WM --> CW
    WM --> PW
    
    MW --> STATE
    QW --> BOUNDS
    PW --> PERSIST
    
    EV1 --> CW
    EV2 --> PW
    EV3 --> PW
    EV4 --> MW
```

### 2. MyComputer - Device Management

```mermaid
graph TB
    subgraph "MyComputer Module"
        MC["MyComputer<br/>(Device Manager)"]
        SESSION["Session Map<br/>(session-id → state)"]
        SHARED["Shared Folders<br/>(local mounts)"]
    end
    
    subgraph "Sidecar Management"
        SM["SidecarManager"]
        SPAWN["Spawn Process<br/>(node sidecar)"]
        MONITOR["Monitor Exit/Error"]
        CLEANUP["Cleanup on Exit"]
    end
    
    subgraph "WebSocket Communication"
        SC["SocketClient<br/>(Socket.io wrapper)"]
        CONNECT["connect(url, auth)"]
        RECONNECT["Auto-reconnect logic<br/>(20 attempts, 3-30s delay)"]
        CONFIG["Receive device_config"]
    end
    
    subgraph "State Management"
        QUEUE["Operation Queue<br/>(sequential)"]
        DEBOUNCE["Debounced Config<br/>Apply (200ms)"]
        RECONNECT_SCHEDULER["Reconnect Scheduler<br/>(exponential backoff)"]
    end
    
    MC --> SESSION
    MC --> SHARED
    MC --> SM
    MC --> SC
    
    SM --> SPAWN
    SM --> MONITOR
    SM --> CLEANUP
    
    SC --> CONNECT
    SC --> RECONNECT
    SC --> CONFIG
    
    MC --> QUEUE
    MC --> DEBOUNCE
    MC --> RECONNECT_SCHEDULER
```

### 3. IPC Interface (Preload Context Bridge)

```mermaid
graph TB
    subgraph "Exposed APIs (via Context Bridge)"
        API["api object<br/>(core IPC)"]
        WINDOW["windowControls<br/>(window management)"]
        STORAGE["localStorage<br/>(custom impl)"]
        AUTH["desktopAuthStorage<br/>(token management)"]
        MC_API["api.myComputer<br/>(device ops)"]
        UPDATE["api.update<br/>(auto-updater)"]
        NAV["api.navigation<br/>(URL tracking)"]
        SHORTCUT["api.shortcut<br/>(keyboard shortcuts)"]
    end
    
    subgraph "Core API Methods"
        CORE1["on/off (event listeners)"]
        CORE2["send (async messages)"]
        CORE3["invoke (RPC calls)"]
        CORE4["openExternal (safe URLs)"]
        CORE5["openChildWindow"]
        CORE6["openPresenterWindow"]
    end
    
    subgraph "Storage Ops"
        STOR1["getItem/setItem"]
        STOR2["removeItem/clear"]
        STOR3["IPC sync via sendSync"]
    end
    
    subgraph "Auth Ops"
        AUTH1["getToken()"]
        AUTH2["setToken(token)"]
        AUTH3["removeToken()"]
    end
    
    API --> CORE1
    API --> CORE2
    API --> CORE3
    API --> CORE4
    API --> CORE5
    API --> CORE6
    
    STORAGE --> STOR1
    STORAGE --> STOR2
    STORAGE --> STOR3
    
    AUTH --> AUTH1
    AUTH --> AUTH2
    AUTH --> AUTH3
    
    WINDOW --> CORE1
    WINDOW --> CORE3
    MC_API --> CORE3
    UPDATE --> CORE3
    NAV --> CORE1
    SHORTCUT --> CORE3
```

---

## Data Flow Diagrams

### Login & Authentication Flow

```mermaid
sequenceDiagram
    participant User as User
    participant Frontend as Frontend UI
    participant Main as Main Process
    participant Backend as Backend API
    participant Storage as Storage
    participant MyComputer as MyComputer

    User->>Frontend: Click login
    Frontend->>Main: open-external (browser URL)
    Main->>Backend: Browser navigate (auth flow)
    
    User->>Backend: Authenticate (OAuth/SSO)
    Backend->>User: Redirect to manus://...?token=XXX&nonce=YYY
    
    Main->>Main: handleProtocolCallback
    Main->>Main: Verify nonce (security check)
    Main->>Storage: Save session token
    Main->>MyComputer: reinit()
    
    MyComputer->>Backend: Connect WebSocket (token auth)
    Backend-->>MyComputer: device_config
    
    Main->>Frontend: Send login-success
    Frontend->>User: Display dashboard
```

### Device Configuration Flow

```mermaid
graph TB
    subgraph "Backend"
        CONFIG_ENDPOINT["Device Config<br/>(REST/WebSocket)"]
    end
    
    subgraph "Socket.io Lifecycle"
        CONNECT["connect<br/>auth: token + deviceId"]
        DEVICE_CONFIG["device_config<br/>event"]
        RECONNECT["Auto-reconnect<br/>on disconnect"]
    end
    
    subgraph "MyComputer Processing"
        PENDING["Pending Config<br/>(stored)"]
        DEBOUNCE["Debounce<br/>200ms"]
        APPLY["Apply Sessions<br/>(enqueue op)"]
        QUEUE["Operation Queue<br/>(serialize)"]
    end
    
    subgraph "Sidecar Spawning"
        FOR_EACH["For each session<br/>in config"]
        SPAWN_SIDECAR["Spawn sidecar<br/>process"]
        SET_FOLDER["Set sharedFolder"]
        MONITOR_PROC["Monitor process"]
    end
    
    CONFIG_ENDPOINT --> CONNECT
    CONNECT --> DEVICE_CONFIG
    DEVICE_CONFIG --> PENDING
    PENDING --> DEBOUNCE
    DEBOUNCE --> APPLY
    APPLY --> QUEUE
    QUEUE --> FOR_EACH
    FOR_EACH --> SPAWN_SIDECAR
    SPAWN_SIDECAR --> SET_FOLDER
    SET_FOLDER --> MONITOR_PROC
    
    MONITOR_PROC -.->|exit/error| RECONNECT
    RECONNECT --> CONNECT
```

---

## Key Components Deep Dive

### Window Manager (windowManager.js)

**Purpose:** Manages lifecycle of all windows (main, quick-ask, child, presenter)

**Key Responsibilities:**
- Create/destroy window instances
- Track window state (singleton vs. multi-instance)
- Emit window lifecycle events
- Send messages to renderer process
- Handle window-closed event cascade

**API:**
```typescript
createWindow(type: 'main' | 'quickAsk', overrides?: object): Window
openNewMainWindow(overrides?: object): MainWindow
openQuickAsk(): QuickAskWindow
mainWindow: MainWindow | null
quickAskWindow: QuickAskWindow | null
broadcast(event: string, data: object): void
```

### MyComputer (myComputer/myComputer.js)

**Purpose:** Manages device sessions, sidecar processes, and WebSocket connection to backend

**Key Responsibilities:**
- Connect to WebSocket backend with auth
- Receive and apply device configurations
- Spawn/manage sidecar child processes
- Track session state (ready, error, connecting)
- Handle reconnection with exponential backoff
- Queue operations for serial execution

**State:**
```typescript
sessions: Map<sessionId, SessionState>
sharedFolders: string[]
isSocketReady: boolean
terminalAvailable: boolean
appliedConfigVersion: number
reconnectStates: Map<sessionId, ReconnectState>
```

**Reconnection Logic:**
- Max 20 reconnect attempts per session
- Initial delay: 3000ms
- Max delay: 30000ms
- Exponential backoff between retries

### Sidecar Manager (myComputer/sidecarManager.js)

**Purpose:** Spawns and manages backend service processes

**Child Process Management:**
- Spawns `sidecar` executable for each shared folder
- Passes WebSocket URL as CLI argument
- Monitors stdin/stdout for errors
- Terminates on exit (SIGTERM, SIGKILL if needed)
- Platform-specific cleanup (PowerShell on Windows, ps on Unix)

**Lifecycle:**
```
spawn() → verify running → return child process
monitor exit → report status → trigger reconnect if unexpected
stop() → SIGTERM → wait 200ms → SIGKILL if needed
```

### IPC Setup (setupIPC.js)

**Main Handlers:**
- `open-external` — safe URL opening (http/https/apple.systempreferences)
- `open-child-window` — spawn new BrowserWindow
- `open-presenter-window` — multi-display presenter mode
- `show-folder-dialog` — native file picker
- `storage-*` — localStorage impl via sendSync
- `token-*` — secure session token storage
- `shortcut-*` — keyboard shortcut management
- `update-*` — electron-updater integration
- `my-computer-reinit` — restart device connection

**Security Features:**
- Context isolation enabled (nodeIntegration: false)
- Sandbox enabled for child windows
- Web security enabled
- Safe URL validation (HTTPS only except localhost)
- Nonce verification on deep links

### Socket Client (myComputer/socketClient.js)

**Socket.io Configuration:**
```typescript
auth: {
    token: sessionId,      // Bearer token
    deviceId: UUID,         // Unique device identifier
    deviceName: string      // macOS computer name
}
transports: ['websocket']
reconnection: true
```

**Key Events:**
- `device_config` — receive device session configuration
- `connect` / `disconnect` — connection lifecycle
- `connect_error` — authentication/network errors
- `error` — socket.io errors

**Connection State:**
```typescript
status: {
    connected: boolean,
    lastError: string | null
}
connectCount: number  // tracks reconnections
```

---

## Frontend Architecture

### Next.js Application Structure

**Entry Points:**
- `/app` — Main dashboard UI
- `/login` — Authentication page
- `/presenter` — Presenter mode UI
- `/quick-ask` — Quick ask dialog
- `404` — Error page

**Build Output:**
- Compiled to `/frontend/out` (static HTML)
- Served via custom Electron protocol handler (`app://manus/...`)
- Next.js export format (static + prerendered)

**Frontend Capabilities (via preload API):**
```typescript
// Window control
windowControls.minimize()
windowControls.maximize()
windowControls.toggleFullscreen()

// Device management
api.myComputer.reinit()

// File system
api.readFileAsArrayBuffer(filePath)
api.showFolderDialog()

// External links
api.openExternal(url)
api.openChildWindow(url)
api.openPresenterWindow(url, data)

// Keyboard shortcuts
api.shortcut.getConfig()
api.shortcut.update(action, accelerator)

// Authentication
desktopAuthStorage.getToken()
desktopAuthStorage.setToken(token)
desktopAuthStorage.removeToken()

// Events
api.on('channel', callback)
api.send('channel', data)
api.invoke('channel', data)
```

---

## Electron Helper Processes

Manus includes 4 helper processes (Frameworks directory):

1. **Manus Helper (GPU).app** — GPU rendering acceleration
2. **Manus Helper (Renderer).app** — Web content rendering delegation
3. **Manus Helper (Plugin).app** — Plugin/extension host
4. **Manus Helper.app** — General-purpose utility process

These follow Electron's multi-process architecture to:
- Isolate crashes (GPU process crash doesn't kill main app)
- Improve performance (offload rendering to dedicated process)
- Sandbox security (each process has limited privileges)

---

## Frameworks & Dependencies

### Core Frameworks
- **Electron Framework** (154MB) — Native Electron runtime
- **ReactiveObjC** — Reactive extensions library
- **Squirrel** — Windows auto-updater backend
- **Mantle** — Model framework (likely not used on macOS)

### Runtime Dependencies
```json
{
  "app-root-path": "^3.1.0",        // CWD resolution
  "electron-log": "^5.4.0",          // Logging
  "electron-updater": "^6.3.9",      // Auto-update
  "electron-window-state": "^5.0.3", // Window persistence
  "lodash-es": "^4.17.21",           // Utility functions
  "sc-prepare-next": "^1.0.3",       // Next.js integration
  "socket.io": "^4.8.1",             // WebSocket server (unused in main)
  "socket.io-client": "^4.8.1",      // WebSocket client
  "ws": "^8.18.0"                    // WebSocket fallback
}
```

---

## Security Architecture

### Context Isolation
```typescript
webPreferences: {
    sandbox: true,
    webSecurity: true,
    nodeIntegration: false,
    contextIsolation: true,
    preload: path.join(__dirname, 'preload.js')
}
```

### Preload Context Bridge
- Exposes minimal API surface to renderer
- All IPC calls go through main process
- No direct Node.js access from renderer

### Protocol Handler Security
```typescript
registerSchemesAsPrivileged([
    {
        scheme: 'app',
        privileges: {
            secure: true,
            standard: true,
            supportFetchAPI: true,
            corsEnabled: true,
            allowServiceWorkers: true,
            stream: true,
            codeCache: true,
        },
    },
]);
```

### URL Validation
```typescript
const SAFE_URLS = [
    /^https?:\/\//,  // HTTP/HTTPS only
    'x-apple.systempreferences:com.apple.preference.security?Privacy_Microphone'
]
```

### Authentication
- Nonce verification on deep-link authentication
- Token stored in keychain (macOS)
- Device ID generated once and persisted
- Socket.io auth with token + deviceId

### CORS Bypass
- Custom CORS header handling (`setupCorsBypass()`)
- Allows frontend to communicate with backend
- Careful about allowing cross-origin requests

---

## Build & Packaging

### Compilation
- **Language:** TypeScript strict mode
- **Compiler:** LLVM Clang (Xcode 15)
- **Target:** macOS 14.5 SDK (Sonoma)
- **Architecture:** arm64 (Apple Silicon) + x86_64 cross-compilation possible

### Signing
- Code signing with `_CodeSignature` directory
- Likely notarized for Gatekeeper
- Asset catalog (Assets.car) for icon resources

### Update Mechanism
- `electron-updater` for delta updates
- `app-update.yml` configuration
- GitHub releases or custom update server
- Auto-check on startup

### Localization
- 40+ language packs (lproj directories)
- Electron native localization support
- Automatic locale detection from system

---

## File Structure

```
Manus.app/
├── Contents/
│   ├── Info.plist                 # App metadata
│   ├── PkgInfo                    # Package type identifier
│   ├── MacOS/
│   │   └── Manus                  # Main executable (arm64)
│   ├── Resources/
│   │   ├── app.asar               # Bundled app (Next.js + dist)
│   │   ├── icon.icns              # App icon
│   │   ├── Assets.car             # Compiled assets
│   │   ├── app-update.yml         # Update configuration
│   │   ├── {lang}.lproj/          # Localization (40+ languages)
│   │   └── bin/                   # Binary resources
│   ├── Frameworks/
│   │   ├── Electron Framework.framework/
│   │   ├── Manus Helper*.app/     # 4 helper processes
│   │   ├── ReactiveObjC.framework/
│   │   ├── Squirrel.framework/
│   │   └── Mantle.framework/
│   └── _CodeSignature/            # Code signing metadata
```

---

## Startup Sequence

```mermaid
graph TD
    A["App Launch"] --> B["Register Single Instance Lock"]
    B -->|Lock acquired| C["app.whenReady()"]
    B -->|Lock failed| Z["Quit"]
    
    C --> D["initLogs<br/>setupCorsBypass<br/>registerAppProtocol"]
    D --> E{"Session Token<br/>exists?"}
    
    E -->|Yes| F["Connect WebSocket"]
    E -->|No| G["Create login window"]
    
    F --> H["Receive device_config"]
    H --> I["Apply session config"]
    I --> J["Spawn sidecars"]
    
    J --> K["Create main window"]
    K --> L["Load Next.js app<br/>app://manus/app"]
    L --> M["Setup IPC handlers<br/>Setup shortcuts<br/>Setup menu<br/>Setup tray"]
    M --> N["App Ready"]
    
    G --> O["User authenticates"]
    O --> P["Deep link callback"]
    P --> F
    
    Z --> Q["Exit"]
    N --> R["Running"]
```

---

## Shutdown Sequence

```mermaid
graph TD
    A["User Closes App"] --> B["app.on('window-all-closed')"]
    B --> C{"mainWindowCount<br/>== 0?"}
    
    C -->|Yes| D["setIsQuitting(true)"]
    C -->|No| E["Wait for other windows"]
    
    D --> F["myComputer.shutdown()"]
    F --> G["Stop all sidecars"]
    G --> H["Disconnect WebSocket"]
    H --> I["Cancel pending operations"]
    I --> J["Clear storage"]
    
    J --> K["Destroy all windows"]
    K --> L["app.quit()"]
    L --> M["Exit Process"]
    
    E --> B
```

---

## Key Environment Variables

```typescript
// From dist/env.js and generated-env.js
envHelper.chatWebsocketUrl  // Backend WebSocket URL
                            // Format: ws://backend/device
                            // Auth via socket.io options

process.platform             // 'darwin' (macOS)
process.pid                  // Parent process ID

// Socket.io query params:
// - token: session ID
// - locale: system locale
// - tz: timezone
// - clientType: 'desktop'
// - clientOS: 'darwin'
// - clientVersion: app version
```

---

## Logging Architecture

### electron-log Integration
- File destination: `~/Library/Application Support/Manus/app.log`
- Console output in development
- Rotation and cleanup handled by electron-log
- Log levels: info, warning, error

### Desktop Logging
```typescript
// From ipcMain handler
ipcMain.on('desktop-log', (_, logs) => {
    electron_log_1.default.info(logs);
});
```

### Sidecar Process Logging
- Logged with session context: `session=<sessionId>`
- Status reporting every 3 seconds (debounced)
- Error conditions logged before reconnect attempts

---

## Error Handling & Recovery

### Socket.io Connection Failures
```
Initial connect → Connection timeout/error
→ Exponential backoff (3s → 30s)
→ Retry up to 20 times
→ Mark session as 'error'
→ Trigger reconnection scheduler
→ User sees "offline" state in UI
```

### Sidecar Process Crashes
```
Process exits unexpectedly
→ reportStatus()
→ Set session status = 'error'
→ scheduleSidecarReconnect()
→ Recreate sidecar after delay
→ Re-apply configuration
```

### Authentication Failures
```
Nonce mismatch on deep-link
→ Send login-error to renderer
→ Clear stored token
→ Prompt user to re-authenticate
```

---

## Performance Optimizations

1. **Debounced Operations** (200ms)
   - Device config application debounced to prevent flapping

2. **Operation Queueing**
   - Operations queued for serial execution
   - Prevents race conditions in device config

3. **Status Reporting** (3s debounce)
   - Aggregates status changes
   - Reduces network traffic

4. **Pre-warming Quick Ask**
   - Frontend hints renderer to pre-render quick-ask window
   - Faster open time when user triggers shortcut

5. **Lazy Loading**
   - Frameworks loaded on-demand (OpenTelemetry, gRPC)
   - Electron protocol handler for static assets

6. **Shared Libraries**
   - lodash-es for tree-shaking
   - Electron framework shared across processes

---

## Comparison with Claude Desktop

| Aspect | Manus | Claude Desktop |
|--------|-------|----------------|
| **Purpose** | Task/project management | AI assistant interface |
| **Backend** | Custom Manus backend | Anthropic API |
| **Device Model** | Sessions with sidecars | Isolated sandboxes |
| **Auth** | Token + device ID | API key + session |
| **Network** | Socket.io WebSocket | HTTP streaming |
| **Frontend** | Next.js (static export) | React + custom UI |
| **Helper Processes** | GPU, Renderer, Plugin | N/A |

---

## Future Architecture Considerations

1. **Plugin System** — Manus Helper (Plugin).app suggests extensibility
2. **Device Management** — Sidecar architecture scales to multiple devices
3. **Offline Mode** — WebSocket reconnection logic supports offline-first patterns
4. **Containerization** — Sidecars could run in containers (Docker/Podman)
5. **Cross-Platform** — Windows and Linux support via Electron platform abstractions

---

## Conclusion

Manus is a sophisticated Electron application that manages a complex system of:
- **UI Layer**: Next.js frontend in isolated renderer process
- **System Integration**: Tight integration with macOS via native frameworks
- **Backend Coordination**: WebSocket-based device management
- **Process Orchestration**: Dynamic spawning and monitoring of sidecar processes
- **State Persistence**: Robust reconnection and error recovery mechanisms

The architecture prioritizes **security** (context isolation, sandbox), **reliability** (operation queueing, exponential backoff), and **performance** (debouncing, lazy loading, multi-process rendering).

---

*Research conducted via static analysis of Manus.app v1.6.2 (June 2024) from extracted app.asar archive. This document reflects the architecture as compiled; source code and design rationale are not available.*
