# CopilotScope — User Guide

## Overview

CopilotScope is a forensic-grade monitor for your VS Code / Copilot AI sessions. It runs invisibly in your system tray and captures every tool call, token flow, system prompt change, and model switch — giving you a complete, tamper-evident audit trail of everything Copilot does on your machine.

---

## Installation

### MSI Installer (Recommended)

1. Download `CopilotScope-1.0.0-setup.exe` from the releases page.
2. Run the setup — **UAC elevation is required** to:
   - Install the self-signed CA certificate into the Windows trust store
   - Register the daemon as an autostart entry
3. The setup will automatically:
   - Install `copilotscoped.exe` (daemon) and `copilotscope.exe` (GUI) to `C:\Program Files\Lackadaisical Security\CopilotScope\`
   - Install the VS Code extension via `code --install-extension`
   - Configure VS Code's HTTP proxy to `http://127.0.0.1:8877`

### Portable EXE

1. Run `CopilotScope-portable.exe` from any location.
2. It will self-extract to `%TEMP%\CopilotScope\` and run `setup.bat`.
3. Portable mode does **not** persist the autostart entry.

---

## First Launch

After installation:

1. The **CopilotScope icon** (👁) appears in your system tray.
2. The VS Code status bar shows `$(eye) CS` at the bottom right.
3. Open VS Code and start a Copilot Chat session — CopilotScope begins capturing immediately.

### CA Certificate Verification

The installer adds a self-signed CA to your Windows Root certificate store named **"CopilotScope Root CA"**. You can verify it at any time:

```powershell
certutil -store Root "CopilotScope Root CA"
```

This certificate is unique to your installation — it is generated fresh during setup and never shared.

---

## Opening the Monitor

- **System tray**: Right-click the 👁 icon → **Open Monitor**
- **VS Code**: Press `Ctrl+Shift+P` → `CopilotScope: Open Monitor`
- **Taskbar**: Double-click the CopilotScope window if already open

---

## Panel Guide

### 🔧 Tool Log

Live feed of every tool call made by Copilot during the session.

| Column | Description |
|--------|-------------|
| Time | When the tool call was observed |
| Tool Name | Name of the tool invoked |
| Source | `builtin`, `mcp`, or `extension` |
| Args (preview) | First 80 characters of the argument JSON |
| Triggered By | `user_action` (green) or `autonomous` / `unknown` (red) |
| Duration | Time until the result was received |

**Click any row** to see the full JSON arguments and result in the detail pane below.

**Filter**: Type in the filter bar to show only entries matching tool name, source, or trigger type.

---

### 📊 Tokens

Compares what VS Code reports vs what CopilotScope actually measures.

- **Table**: One row per turn showing input/output tokens (reported vs actual) and cost
- **Delta highlighting**: Yellow if discrepancy > 5%, red if > 15%
- **Running totals**: Pinned at the bottom of the table

---

### 📋 Sys Prompt

Snapshot viewer for the system prompt that Copilot is using.

- **Turn selector**: Choose any turn from the dropdown to see that turn's snapshot
- **Diff view** (default): Green lines were added since last snapshot; red lines were removed
- **Raw view**: Toggle to see the complete unmodified system prompt
- **Hash**: SHA-256 fingerprint of the prompt — compare across sessions
- **Mutation counter**: "Mutations since session start: N" at the top

---

### 🚨 Alerts

All anomaly alerts, sorted by severity (CRITICAL first) then timestamp.

| Column | Description |
|--------|-------------|
| Severity | 🔴 CRITICAL / 🚩 FLAG / 🟡 WARN / ℹ INFO |
| Time | When the alert fired |
| Rule | Rule ID (e.g. CS-T001) |
| Description | Human-readable explanation |
| Related Event | Link to the triggering event |

**Buttons:**
- **Acknowledge**: Mark as reviewed (stays visible but greyed out)
- **Dismiss**: Remove from the list for this session
- **Filter**: Show only selected severity levels

---

### 🗜 Compact

Tracks context compaction events — when the AI silently summarizes and drops earlier conversation history.

- **Event list**: Each compaction event with tokens before/after and estimated drop
- **Detail pane** (click an event):
  - Tokens before / after / estimated dropped
  - Summary text injected into the context
  - File list before and after (if available)
  - **Was this requested?** — Yes if you initiated it, No if it happened automatically

---

### 🤖 Model

Tracks which AI model is active and when it changes.

- **Unrequested switches**: Count highlighted in red at the top
- **Change log**: All model switches with reason and turn ID
- **Per-model stats**: Turns, tokens, cost, last switch reason

---

### 💰 Price Calc

Calculates the actual cost of your session using current published prices.

- **Per-model breakdown**: Input cost, output cost, cache costs, total
- **Grand total**: Actual (measured) vs reported (from API usage fields)
- **Pricing table date**: When the prices were last updated
- **Check for updates button**: (Future) Fetches current pricing

---

### 📤 Export

Export the full session data as a human-readable or machine-readable file.

**Formats:**
- **JSON (full)**: Complete structured export of all events, alerts, and token ledger
- **Markdown (human)**: Formatted audit report suitable for sharing
- **CSV (token ledger)**: Spreadsheet-friendly token/cost data

**Options:**
- ☑ Include system prompts (uncheck for sensitive sessions)
- ☑ Include tool arguments
- ☐ Include raw HTTP bodies (large; includes full API request/response JSON)

Click **Export** to open a Save File dialog.

---

## Pausing and Resuming Capture

- **System tray** → right-click → **Pause Capture**
- **VS Code** → `Ctrl+Shift+P` → `CopilotScope: Pause`
- The status bar shows `$(eye-closed) CS` when paused

While paused:
- No events are captured or stored
- No alerts fire
- The proxy still intercepts traffic but discards the data

---

## Exporting Session Reports

The Export panel saves data to a file. You can also export directly from the tray menu:

**System tray** → right-click → **Export Session** → choose format → choose save location

The session is **also automatically saved** to `%APPDATA%\CopilotScope\sessions\YYYYMMDD_HHMMSS.db` as a SQLite database throughout the session.

---

## Uninstalling

### Via Windows Settings (MSI install)
1. Settings → Apps → CopilotScope → Uninstall
2. The uninstaller will:
   - Remove the CA certificate from the trust store
   - Unregister the daemon autostart
   - Restore VS Code proxy settings to their previous value
   - Remove the VS Code extension

### Manual cleanup
```powershell
# Remove CA cert
certutil -delstore Root "CopilotScope Root CA"

# Remove VS Code extension
code --uninstall-extension copilotscope-sidecar

# Remove session data (optional)
Remove-Item "$env:APPDATA\CopilotScope" -Recurse
```

---

## Troubleshooting

| Symptom | Solution |
|---------|---------|
| `$(eye) CS` not visible in VS Code status bar | Check Extensions panel — ensure "CopilotScope Sidecar" is installed and enabled |
| No events in Tool Log | Ensure a Copilot Chat turn has been started since the extension activated |
| Alerts panel shows "CS-S003 Proxy Offline" | Check that `cs_proxy.exe` is running (Task Manager → Details). Try restarting the daemon. |
| HTTPS certificate errors in VS Code | Re-run the installer to reinstall the CA certificate, or run `certutil -addstore Root "%APPDATA%\CopilotScope\ca.pem"` |
| Proxy port conflict | Edit `%APPDATA%\CopilotScope\config.json` and change `proxy_port` from `8877` to an available port |
| Session data not saved | Check disk space and permissions on `%APPDATA%\CopilotScope\sessions\` |
