# CopilotScope — Universal Proxy Capture

CopilotScope captures AI API traffic from **any client application**
through its HTTPS intercept proxy (`cs_proxy.exe` / `proxy.js`).

The VS Code extension sidecar is **optional enrichment**, not a
requirement. The proxy alone provides full token-flow visibility across
every supported AI provider.

---

## Supported Clients & Providers

| Client Application | AI Provider | Capture Method |
|--------------------|-------------|----------------|
| VS Code + GitHub Copilot | GitHub Copilot API | Extension sidecar **or** proxy |
| VS Code + other models | OpenAI / Anthropic / Gemini | Proxy |
| **Cursor IDE** | OpenAI / Cursor routing | Proxy |
| **Claude.ai desktop** | Anthropic | Proxy |
| **Claude Code** | Anthropic | Proxy |
| **ChatGPT desktop** | OpenAI | Proxy |
| Windsurf / Codeium | Codeium API | Proxy |
| Amazon Q Developer | AWS CodeWhisperer | Proxy |
| Mistral Le Chat / apps | Mistral AI | Proxy |
| Any OpenAI-compatible app | Any endpoint | Proxy + `add_host` |
| Ollama / LM Studio (local) | Any local model | Proxy (localhost) |
| JetBrains AI Assistant | OpenAI / Anthropic | Proxy |
| Gemini Advanced / AI Studio | Google Gemini | Proxy |

---

## How the Proxy Works

```
┌─────────────────────────────────────────────────────────┐
│                    User's Machine                       │
│                                                         │
│  ┌──────────────────┐      ┌─────────────────────────┐  │
│  │  AI Client App   │─────▶│  CopilotScope Proxy     │  │
│  │  (any provider)  │ HTTP │  127.0.0.1:8877         │  │
│  │                  │CONN  │                         │  │
│  └──────────────────┘      │  • TLS intercept per    │  │
│                            │    provider             │  │
│  ┌──────────────────┐      │  • Request parsing      │  │
│  │  VS Code         │ ─ ─ ▶│  • Token counting       │  │
│  │  + CS Extension  │(opt) │  • Event emission       │  │
│  │  (optional)      │      └────────────┬────────────┘  │
│  └──────────────────┘                   │               │
│                                         │ stdout JSON   │
│  ┌──────────────────────────────────────▼────────────┐  │
│  │         CopilotScope Daemon (copilotscoped.exe)   │  │
│  │  • Anomaly detection  • Session storage (SQLite)  │  │
│  │  • Price calculation  • Named pipe server         │  │
│  └───────────────────────────────┬───────────────────┘  │
│                                  │ IPC                   │
│  ┌───────────────────────────────▼───────────────────┐  │
│  │       CopilotScope GUI (copilotscope.exe)         │  │
│  │  • 8 live panels • Dark mode • Export            │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
         │ HTTPS (validated TLS)
         ▼
    Real AI API
```

---

## System Proxy Configuration

Configure the operating system or the client application to route
HTTPS traffic through `127.0.0.1:8877`.

### Windows — System-wide (all apps)

```powershell
# Set system HTTPS proxy
netsh winhttp set proxy 127.0.0.1:8877

# OR set via registry (Internet Explorer / WinINet settings)
$reg = 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings'
Set-ItemProperty $reg ProxyEnable  1
Set-ItemProperty $reg ProxyServer  '127.0.0.1:8877'

# To revert
netsh winhttp reset proxy
```

### Windows — Per-application (recommended for isolation)

Each AI desktop app can be launched with a proxy override:

```powershell
# Claude.ai desktop
$env:HTTPS_PROXY = 'http://127.0.0.1:8877'
Start-Process 'Claude.exe'

# ChatGPT desktop
$env:HTTPS_PROXY = 'http://127.0.0.1:8877'
Start-Process 'ChatGPT.exe'

# Cursor IDE
$env:HTTPS_PROXY = 'http://127.0.0.1:8877'
Start-Process 'Cursor.exe'
```

### VS Code (per-workspace or user settings)

```json
// settings.json
{
  "http.proxy": "http://127.0.0.1:8877",
  "http.proxyStrictSSL": true
}
```

---

## CA Certificate Trust

The proxy uses a self-signed CA to issue per-host TLS certificates.
The CA must be trusted by the system (or the target app) for HTTPS
interception to succeed without certificate errors.

### Install the CA certificate (Windows)

Run **once** after initial setup — the CopilotScope installer does this
automatically when installed via the MSI:

```powershell
# As Administrator (or via the MSI installer)
$caPath = "$env:ProgramData\CopilotScope\ca.crt"
Import-Certificate -FilePath $caPath `
    -CertStoreLocation Cert:\LocalMachine\Root

Write-Host "CopilotScope CA certificate installed."
```

### Verify the CA is trusted

```powershell
Get-ChildItem Cert:\LocalMachine\Root |
    Where-Object Subject -Like "*CopilotScope*"
```

### Per-application certificate override (without system trust)

Some apps (e.g. Node.js-based tools) respect `NODE_EXTRA_CA_CERTS`:

```powershell
$env:NODE_EXTRA_CA_CERTS = "$env:ProgramData\CopilotScope\ca.crt"
```

---

## Dynamic Host Management

Add new AI providers at runtime via the daemon's pipe interface,
or directly via stdin to the proxy process:

```json
// Add a custom AI API host at runtime
{ "type": "add_host", "host": "api.myprivateai.com" }

// Remove a host
{ "type": "remove_host", "host": "api.myprivateai.com" }

// Pause all capture
{ "type": "pause" }

// Resume capture
{ "type": "resume" }
```

---

## What the Proxy Captures

Every intercepted request and response produces a JSON line on stdout:

### Request event

```json
{
  "type":                "request",
  "request_id":          "uuid-v4",
  "timestamp_us":        1741700000000000,
  "host":                "api.anthropic.com",
  "path":                "/v1/messages",
  "method":              "POST",
  "provider":            "Anthropic",
  "model":               "claude-3-7-sonnet-20250219",
  "model_display":       "Claude 3.7 Sonnet",
  "input_tokens_actual": 4200,
  "tools_count":         3,
  "messages_count":      12,
  "stream":              true,
  "raw_body_size":       16384
}
```

### Response event

```json
{
  "type":              "response",
  "request_id":        "uuid-v4",
  "timestamp_us":      1741700001500000,
  "host":              "api.anthropic.com",
  "provider":          "Anthropic",
  "status_code":       200,
  "model":             "claude-3-7-sonnet-20250219",
  "model_display":     "Claude 3.7 Sonnet",
  "prompt_tokens":     4200,
  "completion_tokens": 850,
  "total_tokens":      5050,
  "finish_reason":     "end_turn",
  "duration_ms":       3240,
  "token_source":      "api_reported"
}
```

---

## Extension vs. Proxy Comparison

| Feature | Proxy only | Proxy + Extension |
|---------|-----------|-------------------|
| Token counts (input/output) | ✅ | ✅ |
| Model name per request | ✅ | ✅ |
| Provider identification | ✅ | ✅ |
| Tool definitions observed | ✅ (from body) | ✅ |
| Tool call classification | ⚠️ body-only | ✅ trigger type |
| Context window utilization | ⚠️ estimated | ✅ from VS Code API |
| System prompt capture | ⚠️ from body | ✅ before each turn |
| Compaction detection | ⚠️ token-delta heuristic | ✅ message-list diff |
| Works with non-VS Code clients | ✅ | N/A |
| Required privilege | User | User |
| Dependency on VS Code API | None | VS Code 1.90+ |

**Legend:** ✅ = full, ⚠️ = partial / estimated, N/A = not applicable.

---

## Privacy & Security

- The proxy only intercepts traffic to the configured AI API hosts.
- Request/response bodies are processed **locally** — never sent anywhere.
- All data is stored in a local SQLite database under `%APPDATA%\CopilotScope\`.
- The CA private key never leaves the local machine.
- Run `copilotscope.exe --clear-session` to delete all stored data.
