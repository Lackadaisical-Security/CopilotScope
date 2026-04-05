# CopilotScope Architecture

## Overview

CopilotScope is a forensic-grade, always-on observer that monitors every interaction between VS Code, GitHub Copilot, and underlying AI API endpoints. It consists of three cooperating native components joined by Win32 named pipes.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      WINDOWS MACHINE                        │
│                                                             │
│  ┌──────────────────────┐    ┌───────────────────────────┐  │
│  │   VS CODE PROCESS    │    │   COPILOT SCOPE DAEMON    │  │
│  │                      │    │   (C++  Win32 service)    │  │
│  │  ┌────────────────┐  │    │                           │  │
│  │  │ CopilotScope   │◄─┼────┤  Named Pipe Server        │  │
│  │  │ VS Code Ext    │  │    │  ├─ Process watcher       │  │
│  │  │ (TypeScript)   │─►┼────┤  ├─ ETW network tap       │  │
│  │  └────────────────┘  │    │  ├─ Token counter (C++)   │  │
│  │                      │    │  ├─ Price calculator       │  │
│  │  Intercepts:         │    │  ├─ Anomaly detector       │  │
│  │  · vscode.lm.*       │    │  └─ Alert triage engine   │  │
│  │  · chat participants │    │                           │  │
│  │  · tool invocations  │    └──────────┬────────────────┘  │
│  │  · context window    │               │ Named Pipe         │
│  └──────────────────────┘               │                   │
│                                         ▼                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │               COPILOTSCOPE GUI  (C++ Win32)          │   │
│  │                                                      │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │   │
│  │  │ Tool Log │ │ Tokens   │ │ Sys Prom │ │ Alerts │  │   │
│  │  │ Live     │ │ Real vs  │ │ Diff     │ │ Red    │  │   │
│  │  │ Feed     │ │ Reported │ │ Viewer   │ │ Flags  │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────┘  │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │   │
│  │  │ Compact  │ │ Model    │ │ Price    │ │ Export │  │   │
│  │  │ Events   │ │ Tracker  │ │ Calc     │ │ Report │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────┐                       │
│  │   HTTPS INTERCEPT PROXY          │                       │
│  │   (Node.js, runs as child proc)  │                       │
│  │   Listens: 127.0.0.1:8877        │                       │
│  │   Captures: api.githubcopilot.com│                       │
│  │             *.openai.azure.com   │                       │
│  │             *.anthropic.com      │                       │
│  └──────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Responsibilities

### 1. CopilotScope Daemon (`copilotscoped.exe`)

The central nervous system. Runs as a tray application with no visible window.

| Subsystem | File | Responsibility |
|-----------|------|----------------|
| Pipe Server | `pipe_server.cpp` | Receives events from extension + proxy via `\\.\pipe\CopilotScope`; sends to GUI via `\\.\pipe\CopilotScopeGUI` |
| Token Counter | `token_counter.cpp` | BPE-approximation token counting for all model families |
| Price Calculator | `price_calc.cpp` | Calculates USD cost per request from `pricing_table.json` |
| Anomaly Engine | `anomaly_engine.cpp` | Evaluates all 15+ CS-XXXX alert rules in real time |
| Alert Triage | `alert_triage.cpp` | Routes fired alerts to SQLite + GUI broadcast |
| Session Store | `session_store.cpp` | Persists all events to SQLite in `%APPDATA%\CopilotScope\sessions\` |
| Process Watcher | `process_watcher.cpp` | Monitors VS Code process lifecycle; detects restarts |
| ETW Tap | `etw_tap.cpp` | Event Tracing for Windows — network event correlation |
| Main | `main.cpp` | Singleton mutex, tray icon, child process lifecycle, shutdown |

**Named pipes created:**

| Pipe | Direction | Clients |
|------|-----------|---------|
| `\\.\pipe\CopilotScope` | inbound | VS Code Extension, HTTPS Proxy |
| `\\.\pipe\CopilotScopeGUI` | outbound | GUI window |

### 2. VS Code Extension (`copilotscope-sidecar`)

TypeScript extension running inside the VS Code Extension Host. Has direct access to `vscode.lm.*` APIs.

| Module | File | Responsibility |
|--------|------|----------------|
| Entry Point | `extension.ts` | Activation, pipe connect, command registration, deactivation |
| LM Interceptor | `lm_interceptor.ts` | Monkey-patches `vscode.lm.sendChatRequest` and `selectChatModels` |
| Tool Watcher | `tool_watcher.ts` | Captures `LanguageModelToolCallPart` / `LanguageModelToolResultPart` |
| Context Snapshot | `context_snapshot.ts` | SHA-256 hashes system prompt; estimates actual token counts |
| Compact Detector | `compact_detector.ts` | Detects context compaction via 5 heuristics |
| Model Tracker | `model_tracker.ts` | Detects model identity changes; classifies change reasons |
| Pipe Client | `pipe_client.ts` | Connects to `\\.\pipe\CopilotScope`; backoff reconnect; message buffer |

### 3. HTTPS Intercept Proxy (`cs_proxy.exe`)

Node.js application compiled to a standalone EXE via `pkg`. Performs HTTPS MITM interception.

| Module | File | Responsibility |
|--------|------|----------------|
| Proxy Server | `proxy.js` | HTTP/1.1 CONNECT tunnel intercept, TLS re-encryption |
| Cert Generator | `cert_gen.js` | Self-signed CA + per-host leaf cert generation |
| Request Parser | `request_parser.js` | Parses OpenAI, Anthropic, Gemini, Copilot API payloads |

Communicates with the daemon via **stdout JSON-lines** (events) and **stdin** (commands).

### 4. GUI (`copilotscope.exe`)

Standalone Win32 window with 8 tabbed panels. Connects to `\\.\pipe\CopilotScopeGUI` to receive live events.

| Panel | Class | Display |
|-------|-------|---------|
| Tool Log | `PanelToolLog` | Virtual list; color-coded by trigger type |
| Tokens | `PanelTokens` | Reported vs actual token comparison table |
| Sys Prompt | `PanelSysPrompt` | Snapshot diff viewer with SHA-256 display |
| Alerts | `PanelAlerts` | Severity-sorted alert list with acknowledge/dismiss |
| Compact | `PanelCompact` | Compaction timeline and detail view |
| Model | `PanelModel` | Model change history and unrequested switch count |
| Price Calc | `PanelPrice` | Per-model and session cost breakdown |
| Export | `PanelExport` | JSON/Markdown/CSV export with format options |

---

## Data Flow: User Types a Message

```
Step 1  User types in VS Code Chat panel and presses Enter
        ↓
Step 2  vscode.lm.sendChatRequest() is called by Copilot
        ↓ (intercepted by lm_interceptor.ts)
Step 3  Extension sends MSG_TURN_START to daemon (turn_id, model, message count)
        ↓
Step 4  ContextSnapshot hashes system prompt, estimates token count
        Extension sends MSG_SYSPROMPT_SNAPSHOT (if changed) + MSG_CONTEXT_UPDATE
        ↓
Step 5  CompactDetector checks for compaction signals
        (optionally sends MSG_COMPACT_DETECTED)
        ↓
Step 6  Original sendChatRequest() called → request sent to API
        ↓
Step 7  HTTPS Proxy captures the raw HTTP request body
        Proxy emits JSON line: { type: 'request', model, input_tokens_actual, ... }
        ↓
Step 8  Daemon receives proxy stdout line → wraps as MSG_HTTP_REQUEST
        ↓
Step 9  Response stream arrives at extension
        For each LanguageModelToolCallPart: ToolWatcher sends MSG_TOOL_CALL
        ↓
Step 10 Response stream finishes → MSG_TURN_END sent
        HTTPS Proxy captures response body → emits { type: 'response', tokens, cost }
        ↓
Step 11 Daemon:
        a. Enriches events with token counts + price calculation
        b. Runs AnomalyEngine on each event → fires alerts if rules match
        c. Persists everything to SQLite session DB
        d. Broadcasts enriched events to GUI via CopilotScopeGUI pipe
        ↓
Step 12 GUI receives events → dispatches to appropriate panel
        Alerts panel updates, token ledger row added, tool log entry added
```

---

## Security Model

| Concern | Design Decision |
|---------|----------------|
| CA Certificate | Generated locally per installation, unique key pair. Never leaves the machine. |
| Data Storage | All session data in `%APPDATA%\CopilotScope\` — no cloud sync, no telemetry. |
| Elevation | Required only at install time for CA cert trust store write. Daemon runs as standard user. |
| Sensitive Data | System prompts and tool arguments stored locally in SQLite. Export warns before writing. |
| Network Proxy | Intercepts only listed AI API hosts; all other traffic is tunneled transparently. |

---

## Threading Model

### Daemon

```
Main thread          — tray message pump (wWinMain)
Pipe accept thread   — ConnectNamedPipe loop for CopilotScope pipe
GUI pipe thread      — ConnectNamedPipe loop for CopilotScopeGUI pipe
Per-client threads   — one read thread per connected client
Proxy stdout thread  — reads JSON lines from proxy child process
Watcher thread       — process enumeration (CreateToolhelp32Snapshot loop)
```

### GUI

```
Main thread (UI)     — WndProc, tab switching, GDI paint
Pipe client thread   — overlapped read loop on CopilotScopeGUI
                       Posts WM_APP+N messages to main thread queue
```

---

## Build Dependencies

| Component | Dependency | Notes |
|-----------|-----------|-------|
| Daemon | SQLite 3 amalgamation | Statically linked, `third_party/sqlite3/` |
| Daemon | shlwapi, advapi32 | Windows SDK |
| Daemon | wbemuuid, ole32, oleaut32 | WMI for process watcher |
| GUI | comctl32, uxtheme, gdiplus | Common Controls v6, GDI+ charts |
| GUI | comdlg32, shlwapi | File dialogs |
| Extension | `vscode` API | Bundled; requires VS Code ≥ 1.85.0 |
| Proxy | `node-forge` | TLS cert generation |
| Proxy | `ws` | WebSocket support (future) |

---

## Source Tree

```
copilotscope/
  src/
    daemon/      — C++ Win32 background service
    gui/         — C++ Win32 GUI application
    extension/   — VS Code Extension (TypeScript)
    proxy/       — HTTPS intercept proxy (Node.js)
  shared/        — Protocol headers + pricing table
  packaging/     — WiX MSI + sign scripts
  tests/         — Unit tests (C++/Jest) + integration (PowerShell)
  docs/          — This documentation
  third_party/   — SQLite amalgamation
```
