# CopilotScope — IPC Protocol Reference

## Overview

All CopilotScope components communicate via Win32 named pipes using a binary header followed by a UTF-8 JSON payload.

**Pipe names:**

| Pipe | Direction |
|------|-----------|
| `\\.\pipe\CopilotScope` | Extension → Daemon, Proxy → Daemon |
| `\\.\pipe\CopilotScopeGUI` | Daemon → GUI |

---

## Wire Format

Every message on the pipe consists of a **fixed 40-byte header** immediately followed by `payload_len` bytes of UTF-8 JSON.

### Header (C struct, 1-byte packed)

```c
#pragma pack(push, 1)
typedef struct {
    uint32_t magic;          // 0xC0511337  — used to detect misaligned reads
    uint32_t version;        // Currently 1
    uint32_t msg_type;       // See MsgType enum below
    uint32_t payload_len;    // Length of the JSON payload in bytes (0 = no payload)
    uint64_t timestamp_us;   // Microseconds since Unix epoch (UTC)
    uint8_t  session_id[16]; // Random 128-bit session UUID (raw bytes, not hex)
} CsMessageHeader;           // sizeof = 40
#pragma pack(pop)
```

### Byte layout

```
Offset  Size  Field
──────  ────  ─────────────────────────────────────
0       4     magic         = 0x37 0x13 0x51 0xC0 (LE)
4       4     version       = 0x01 0x00 0x00 0x00 (LE)
8       4     msg_type      (LE uint32)
12      4     payload_len   (LE uint32)
16      8     timestamp_us  (LE uint64)
24      16    session_id    (raw bytes)
40      N     UTF-8 JSON payload  (N = payload_len)
```

**All multi-byte integers are little-endian.**

---

## Message Types

```c
typedef enum {
    // Extension → Daemon
    MSG_TOOL_CALL          = 0x0001,
    MSG_TOOL_RESULT        = 0x0002,
    MSG_SYSPROMPT_SNAPSHOT = 0x0003,
    MSG_CONTEXT_UPDATE     = 0x0004,
    MSG_COMPACT_DETECTED   = 0x0005,
    MSG_MODEL_CHANGE       = 0x0006,
    MSG_TURN_START         = 0x0007,
    MSG_TURN_END           = 0x0008,
    MSG_LM_REQUEST_RAW     = 0x0009,
    MSG_LM_RESPONSE_RAW    = 0x000A,

    // Proxy → Daemon
    MSG_HTTP_REQUEST       = 0x0100,
    MSG_HTTP_RESPONSE      = 0x0101,

    // Daemon → GUI
    MSG_EVENT_BUNDLE       = 0x0200,
    MSG_ALERT              = 0x0201,
    MSG_STATS_UPDATE       = 0x0202,
    MSG_SESSION_SUMMARY    = 0x0203,

    // GUI → Daemon
    MSG_EXPORT_REQUEST     = 0x0300,
    MSG_CLEAR_SESSION      = 0x0301,
    MSG_PAUSE_CAPTURE      = 0x0302,
    MSG_RESUME_CAPTURE     = 0x0303,
} MsgType;
```

---

## Payload Schemas

### MSG_TOOL_CALL (0x0001)

Sent by the extension when a `LanguageModelToolCallPart` appears in the response stream.

```json
{
  "call_id":      "uuid-v4",
  "tool_name":    "create_file",
  "tool_source":  "builtin | mcp | extension",
  "arguments":    { },
  "turn_id":      "uuid-v4",
  "triggered_by": "user_action | autonomous | unknown",
  "stack_hint":   "optional string — what triggered this call"
}
```

### MSG_TOOL_RESULT (0x0002)

```json
{
  "call_id":     "uuid-v4",
  "turn_id":     "uuid-v4",
  "result":      { },
  "duration_us": 0,
  "error":       "optional error string"
}
```

### MSG_SYSPROMPT_SNAPSHOT (0x0003)

Sent when the system prompt hash changes between turns.

```json
{
  "snapshot_id":          "uuid-v4",
  "turn_id":              "uuid-v4",
  "prompt_hash":          "sha256 hex string (64 chars)",
  "prompt_text":          "full system prompt content",
  "token_count_reported": 0,
  "token_count_actual":   0,
  "delta_from_last":      "diff string or null"
}
```

### MSG_CONTEXT_UPDATE (0x0004)

Sent once per turn start.

```json
{
  "turn_id":              "uuid-v4",
  "tokens_used_reported": 0,
  "tokens_max_reported":  128000,
  "tokens_used_actual":   0,
  "pct_reported":         0.0,
  "pct_actual":           0.0,
  "discrepancy_pct":      0.0
}
```

### MSG_COMPACT_DETECTED (0x0005)

```json
{
  "compact_id":               "uuid-v4",
  "turn_id":                  "uuid-v4",
  "trigger":                  "threshold | manual | unknown",
  "context_before_tokens":    0,
  "context_after_tokens":     0,
  "dropped_tokens_estimated": 0,
  "summary_injected":         "summary text",
  "summary_token_count":      0,
  "files_in_context_before":  [],
  "files_in_context_after":   [],
  "was_requested":            false
}
```

### MSG_MODEL_CHANGE (0x0006)

```json
{
  "from_model":    "gpt-4o or null",
  "to_model":      "claude-sonnet-4-5",
  "change_reason": "user_selected | automatic | fallback | unknown",
  "was_requested": false,
  "turn_id":       "uuid-v4"
}
```

### MSG_TURN_START (0x0007)

```json
{
  "turn_id":       "uuid-v4",
  "model_id":      "gpt-4o",
  "message_count": 5,
  "user_text_len": 142,
  "has_tools":     true,
  "tool_count":    3,
  "timestamp_us":  0
}
```

### MSG_TURN_END (0x0008)

```json
{
  "turn_id":        "uuid-v4",
  "finish_reason":  "stop",
  "output_text_len": 0,
  "tool_calls_made": 0,
  "duration_ms":    0,
  "first_token_ms": 0
}
```

### MSG_ALERT (0x0201)

Sent by daemon to GUI when an anomaly rule fires.

```json
{
  "alert_id":        "uuid-v4",
  "severity":        "INFO | WARN | FLAG | CRITICAL",
  "rule_id":         "CS-T001",
  "rule_name":       "Autonomous Tool Call",
  "description":     "Tool 'create_file' called autonomously (no user prompt in last 2s)",
  "related_event_id": "uuid-v4 or null",
  "timestamp_us":    0,
  "auto_dismissed":  false
}
```

### MSG_STATS_UPDATE (0x0202)

Sent periodically by daemon to GUI (≈ every 2 seconds during active session).

```json
{
  "session_id":          "hex string",
  "requests":            0,
  "total_input_tokens":  0,
  "total_output_tokens": 0,
  "total_cost_usd":      0.0,
  "alerts_critical":     0,
  "alerts_flag":         0,
  "alerts_warn":         0,
  "alerts_info":         0,
  "current_model":       "gpt-4o",
  "context_pct_actual":  0.0
}
```

### MSG_EXPORT_REQUEST (0x0300)

Sent by GUI → Daemon to trigger a session export.

```json
{
  "format":               "json | markdown | csv",
  "include_system_prompts": true,
  "include_tool_args":      true,
  "include_raw_http":       false,
  "output_path":            "C:\\Users\\...\\export.json"
}
```

---

## Example Wire Message (hex dump)

`MSG_TURN_START` with payload `{"turn_id":"abc","model_id":"gpt-4o",...}`:

```
37 13 51 C0   ← magic (LE)
01 00 00 00   ← version = 1
07 00 00 00   ← msg_type = MSG_TURN_START (0x0007)
2A 00 00 00   ← payload_len = 42
xx xx xx xx  xx xx xx xx   ← timestamp_us (8 bytes LE)
xx xx xx xx  xx xx xx xx  xx xx xx xx  xx xx xx xx   ← session_id (16 bytes)
7B 22 74 75  ...           ← UTF-8 JSON payload (42 bytes)
```

---

## Versioning

The `version` field in the header is currently `1`. Future breaking changes to the header layout will increment this value. Receivers **MUST** drop messages with an unrecognized version rather than attempting to parse.

Payload schema changes within the same version are additive only (new optional fields). Parsers should ignore unknown JSON keys.

---

## Implementation Notes

- The receiver must buffer incoming bytes until `payload_len` bytes have arrived after the header.
- Maximum payload: 4 MB (`CS_MAX_PAYLOAD = 4 * 1024 * 1024`). Larger payloads are rejected.
- The `session_id` in the header is the same 16-byte UUID for all messages from a single extension activation.
- The `timestamp_us` is set by the **sender** at message construction time.
