# Changelog

All notable changes to CopilotScope will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.1.0] — 2026-03-19

### Added

#### Model & Provider Support (March 2026)
- **GPT-5 series**: GPT-5.4, GPT-5.3, GPT-5.2, GPT-5.1, GPT-5, GPT-5 mini
- **GPT-4.1**: New GPT-4 family model with 1M token context
- **Claude 4.6 series**: Claude Opus 4.6, Claude Sonnet 4.6
- **Claude Haiku 4.5**: Cost-efficient Claude 4 tier
- **Gemini 3.1 series**: Gemini 3.1 Pro, Gemini 3.1 Flash, Gemini 3.1 Flash Lite
- **Grok-3**: xAI Grok-3 and Grok-3 mini
- **DeepSeek V3 & R1**: DeepSeek chat and reasoning models
- **Codestral**: Mistral's code-focused model
- **5 new AI providers**: xAI/Grok, DeepSeek, Together AI, Perplexity, Mistral

#### Proxy Enhancements
- **Gemini request parsing**: Full support for `contents` format, `system_instruction`,
  model extraction from URL path, `function_declarations` tool format
- **Gemini response parsing**: `candidates`/`usageMetadata` format with `functionCall` detection
- **Cache token tracking**: Anthropic `cache_read_input_tokens`/`cache_creation_input_tokens`,
  OpenAI `prompt_tokens_details.cached_tokens`, Gemini `cachedContentTokenCount`
- **Tool call counting**: Count tool calls in responses across all providers
- **Gzip/deflate/brotli decompression**: Auto-decompress response bodies before parsing
- **`max_completion_tokens`**: Support for o3/o1 reasoning model format
- **`tool_names` array**: Emit list of tool names with each request event
- **`has_system_prompt`**: Flag whether request contains a system prompt

#### Extension Enhancements
- **Expanded built-in tool list**: 35+ known VS Code/Copilot tools including git, testing,
  web search, IDE operations (up from 15)
- **Additional compaction markers**: 11 detection patterns for modern AI agent context management
- **Comprehensive model tiers**: 30+ models mapped across 5 capability tiers for downgrade detection

#### Tests
- **29 new proxy tests** (73 total): Gemini format, cache tokens, tool calls, gzip, new providers
- **25 new extension tests** (48 total): tool_watcher, context_snapshot with full coverage

### Changed
- Pricing table updated to March 2026 rates for all 30+ models
- Model tracker tiers expanded for GPT-5.x, Claude 4.6, Gemini 3.1, Grok, DeepSeek families

---

## [1.0.0] — 2026-03-18

### Added

#### Core Components
- **Daemon** (`copilotscoped.exe`) — C++20 Win32 background service with tray icon
  - Named pipe server for extension and proxy communication
  - Named pipe client for GUI event forwarding
  - Process watcher (WMI) for VS Code lifecycle detection
  - ETW network tap for AI API connection monitoring
  - Token counter with per-model-family BPE approximation
  - Price calculator with live pricing table (`pricing_table.json`)
  - Anomaly engine with 16 detection rules (CS-T001 through CS-S003)
  - Alert triage with severity classification (INFO/WARN/FLAG/CRITICAL)
  - Session store (SQLite WAL mode, statically linked amalgamation)

- **GUI** (`copilotscope.exe`) — C++20 Win32 tabbed monitor
  - 8 tabbed panels: Tool Log, Tokens, Sys Prompt, Alerts, Compact, Model, Price Calc, Export
  - Live session statistics in status bar (requests, tokens, cost, uptime, alerts)
  - Real-time pipe client with automatic reconnect
  - Pause/Resume capture control

- **HTTPS Intercept Proxy** (`cs_proxy.exe`) — Node.js 20 standalone binary
  - Universal capture for all AI service providers
  - Supported providers: GitHub Copilot, OpenAI, Anthropic/Claude, Azure OpenAI,
    Google Gemini, Cursor, Windsurf/Codeium, Amazon Q, Mistral AI, Groq, Ollama
  - Per-host TLS MITM with generated CA certificate
  - Streaming response support (SSE)
  - Token counting from request/response bodies
  - Stdin/stdout JSON-lines IPC with daemon

- **VS Code Extension Sidecar** (`copilotscope-sidecar-1.0.0.vsix`) — TypeScript
  - Optional enrichment for VS Code users
  - LM API interceptor (`vscode.lm.*` hooks)
  - Tool call watcher with trigger classification (user/autonomous)
  - Context window snapshot with system prompt diffing
  - Compaction detector (history length drop, summary injection)
  - Model tracker with change reason detection

#### Theme System
- **6 built-in themes** with full color palette (22 color fields each):
  - **Light** — clean white with system grays
  - **Dark** — charcoal with off-white text
  - **Cyber 80s** — retro magenta/cyan on deep black, neon lime accents
  - **Terminal 90s** — warm amber on black with phosphor green accents
  - **Matrix** — bright phosphor green on jet black
  - **Custom** — fully user-defined, persisted to Windows registry
- Theme selector combobox in toolbar with live preview
- Custom theme editor with color picker for background, foreground, and accent
- Automatic palette derivation (row colors, severity colors, status bar) from base colors
- Dark/light mode detection via background luminance calculation

#### Alert Rules
- **Tool Rules:** CS-T001 (autonomous tool call), CS-T002 (filesystem write),
  CS-T003 (deep chain), CS-T004 (unlisted tool), CS-T005 (destructive command),
  CS-T006 (background process)
- **Context Rules:** CS-C001 (unrequested compact), CS-C002 (token discrepancy),
  CS-C003 (system prompt mutation), CS-C004 (prompt growth), CS-C005 (context reset)
- **Model Rules:** CS-M001 (unrequested switch), CS-M002 (fallback), CS-M003 (upgrade)
- **Network Rules:** CS-N001 (unlisted endpoint), CS-N002 (oversized request),
  CS-N003 (PII pattern), CS-N004 (token estimate mismatch)
- **System Rules:** CS-S001 (extension host crash), CS-S002 (VS Code restart),
  CS-S003 (proxy offline)

#### Packaging
- WiX 3.14 MSI installer with CA cert installation, autostart registry, VS Code extension install
- Portable EXE option (self-extracting, no installation required)
- Static MSVC runtime — zero runtime dependencies
- MinGW-w64 cross-compilation toolchain for Linux CI builds

#### Pricing Table
- Seed data for 14 models across OpenAI, Anthropic, and Google
- Per-model fields: input/output/cache-read/cache-write per 1M tokens
- Hot-reloadable JSON format with update timestamps

#### IPC Protocol
- Binary wire format: 40-byte header + UTF-8 JSON payload
- Magic number validation (`0xC0511337`) with version check
- 14 message types covering extension → daemon, proxy → daemon, daemon → GUI, GUI → daemon
- 4 MB payload hard cap with 64 KB pipe buffers

#### Tests
- **Proxy tests** (44 tests) — request parser, provider config, cert gen, token estimation
- **Extension tests** (23 tests) — LM interceptor, compact detector, model tracker
- **Daemon C++ tests** — anomaly engine, pipe server, price calc, token counter
- **Integration tests** — PowerShell E2E harness with mock VS Code session

#### Documentation
- Architecture overview (`docs/architecture.md`)
- IPC protocol reference (`docs/ipc_protocol.md`)
- Alert rules reference (`docs/alert_rules.md`)
- Pricing table format (`docs/pricing_table.md`)
- User guide (`docs/user_guide.md`)
- Developer setup (`docs/dev_setup.md`)
- Screenshots (`docs/screenshots.md`)
- Full implementation plan (`COPILOT_MONITOR_IMPLEMENTATION_PLAN_PROMPT.md`)

### Changed
- Daemon event dispatcher now parses `turn_id` from JSON payloads for full event correlation

### Fixed
- Removed build zip archives from repository (extracted into proper directories)

---

## [Unreleased]

### Planned
- Linux native build support
- Additional real-time statistics in GUI panels
- Session comparison and historical analysis
- Plugin interface for custom anomaly rules
