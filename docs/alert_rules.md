# CopilotScope — Alert Rules Reference

## Severity Levels

| Level | Icon | Meaning |
|-------|------|---------|
| `CRITICAL` | 🔴 | Security concern, potential data exfiltration, auth anomaly |
| `FLAG` | 🚩 | Behavior happened without being explicitly asked |
| `WARN` | 🟡 | Unusual but not necessarily malicious |
| `INFO` | ℹ️ | Noteworthy event, logged for audit trail |

---

## Alert Rule Definitions

### Tool Call Rules (CS-T)

---

#### CS-T001 — Autonomous Tool Call
**Severity:** FLAG  
**Trigger:** A tool call is observed more than 2 seconds after the last `MSG_TURN_START`, or when `triggered_by = "autonomous"`.  
**Rationale:** Tool calls should be a direct consequence of user messages. Calls that happen long after the last user prompt, or in absence of one, suggest the model is acting without explicit instruction.  
**How to dismiss:** If you intentionally started a long autonomous agent workflow, acknowledge this alert.

---

#### CS-T002 — Autonomous File System Write
**Severity:** FLAG  
**Trigger:** A file-writing tool (`create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `str_replace_editor`) is called with `triggered_by = "autonomous"`.  
**Rationale:** Writing files without a user prompt is a significant autonomous action that should always be explicit.  
**How to dismiss:** Verify the file content is what you intended.

---

#### CS-T003 — Deep Tool Chain
**Severity:** WARN  
**Trigger:** More than 10 tool calls occur within a single turn (since the last `MSG_TURN_START`).  
**Rationale:** Deep tool chains can accumulate unintended side effects. Review what was done.  
**How to dismiss:** Review the Tool Log panel for the full chain.

---

#### CS-T004 — Tool Not in Last Known Definitions
**Severity:** WARN  
**Trigger:** A tool is called whose name was not present in the `options.tools` array of the most recent LM request.  
**Rationale:** Tools appearing from outside the declared set may indicate MCP injection or dynamic tool registration that wasn't anticipated.

---

#### CS-T005 — Destructive Terminal Command
**Severity:** CRITICAL  
**Trigger:** A terminal/shell tool (`run_in_terminal`, `run_terminal_cmd`, `bash`, `execute_command`) is called with arguments containing any of:
- `rm -rf`
- `rmdir /s`
- `--force` (in a destructive context)
- `git reset --hard`
- `DROP TABLE`
- `DELETE FROM`
- `format c:`
- `del /f /s`  

**Rationale:** These commands are irreversible and high-impact.  
**How to dismiss:** Verify the command is intentional before acknowledging.

---

#### CS-T006 — Background Process Spawned Without Request
**Severity:** FLAG  
**Trigger:** A terminal tool is called with `isBackground: true` and the user's message text did not contain keywords like "background", "server", "watch", "daemon", or "start".  
**Rationale:** Background processes persist beyond the current turn and can represent a foothold.

---

### Context / Compaction Rules (CS-C)

---

#### CS-C001 — Unrequested Context Compaction
**Severity:** FLAG  
**Trigger:** `MSG_COMPACT_DETECTED` received with `was_requested = false`.  
**Rationale:** Compaction should only happen when the user explicitly says "compact" or initiates it via the UI. Silent compaction means context was silently dropped.  
**How to dismiss:** If you intended the compaction, click Acknowledge.

---

#### CS-C002 — Context Window Discrepancy
**Severity:** WARN  
**Trigger:** The reported context percentage and the actual calculated percentage differ by more than 10 percentage points.  
**Rationale:** The reported context gauge may be misleading. This alert shows when the discrepancy is significant enough to affect decision-making.

---

#### CS-C003 — System Prompt Changed Without User Edit
**Severity:** WARN  
**Trigger:** `MSG_SYSPROMPT_SNAPSHOT` reports a non-null `delta_from_last` value and the change was not initiated by the user opening VS Code settings.  
**Rationale:** System prompt changes between turns can indicate instruction injection or unintended prompt mutation.

---

#### CS-C004 — System Prompt Grew Significantly
**Severity:** INFO  
**Trigger:** The system prompt token count increased by more than 20% in a single turn.  
**Rationale:** Rapid prompt growth may indicate injection of large context blocks without user knowledge.

---

#### CS-C005 — Context Cleared Without User Action
**Severity:** FLAG  
**Trigger:** `MSG_CONTEXT_UPDATE` shows `tokens_used_actual` drops to less than 5% of the previous value, and no `MSG_CLEAR_SESSION` was received from the GUI.  
**Rationale:** Silent context resets discard conversation history without operator knowledge.

---

### Model Rules (CS-M)

---

#### CS-M001 — Model Changed Without User Request
**Severity:** FLAG  
**Trigger:** `MSG_MODEL_CHANGE` received with `was_requested = false`.  
**Rationale:** Model switches should be explicit. An automatic switch to a different model (especially a weaker one) is a transparency concern.  
**How to dismiss:** If you are aware the switch was intentional (e.g. a configured fallback), acknowledge this alert.

---

#### CS-M002 — Fallback to Weaker Model
**Severity:** WARN  
**Trigger:** A model change is detected where the `to_model` has a lower capability tier than the `from_model` per the internal tier table.  
**Rationale:** Fallback to a less capable model means your requests may be processed by a model you didn't choose.

---

#### CS-M003 — Model Upgraded Mid-Session
**Severity:** INFO  
**Trigger:** A model change is detected where the `to_model` has a higher capability tier than `from_model`.  
**Rationale:** Informational — you may be incurring higher costs than expected if the upgrade happened automatically.

---

### Network Rules (CS-N)

---

#### CS-N001 — Request to Unlisted API Endpoint
**Severity:** FLAG  
**Trigger:** The HTTPS proxy captures a request to a host not in the approved list:
- `api.githubcopilot.com`
- `copilot-proxy.githubusercontent.com`
- `*.openai.azure.com`
- `api.openai.com`
- `api.anthropic.com`
- `generativelanguage.googleapis.com`
- `*.azure-api.net`

**Rationale:** Requests to unknown AI endpoints may indicate exfiltration or use of unauthorized services.

---

#### CS-N002 — Unusually Large Request Body
**Severity:** WARN  
**Trigger:** Captured request body exceeds 500 KB.  
**Rationale:** Very large requests may be sending more context than intended, including files or data that should not be shared.

---

#### CS-N003 — Possible PII in Response
**Severity:** CRITICAL  
**Trigger:** Response body matches heuristic patterns for:
- Social Security Numbers (`\d{3}-\d{2}-\d{4}`)
- Credit card numbers (Luhn-valid 16-digit sequences)
- Private key headers (`-----BEGIN RSA PRIVATE KEY-----`)
- AWS access key patterns (`AKIA[A-Z0-9]{16}`)  

**Note:** This is a heuristic only and may produce false positives. It is **not** a DLP solution.  
**Rationale:** Flags potential credential or PII leakage in model responses.

---

#### CS-N004 — Token Count Discrepancy
**Severity:** INFO  
**Trigger:** The API-reported token count differs from the local BPE estimate by more than 20%.  
**Rationale:** Large discrepancies may indicate the model is processing more context than the extension can observe (e.g. injected hidden context).

---

### System Rules (CS-S)

---

#### CS-S001 — Extension Host Restarted Unexpectedly
**Severity:** WARN  
**Trigger:** The VS Code extension host process exits and restarts without a corresponding user-initiated VS Code restart.  
**Rationale:** Unexpected extension host restarts can cause gaps in the event log.

---

#### CS-S002 — New VS Code Session
**Severity:** INFO  
**Trigger:** A new `Code.exe` process is detected (VS Code restarted).  
**Rationale:** Session continuity note — events before the restart belong to a different session.

---

#### CS-S003 — Proxy Capture Offline
**Severity:** WARN  
**Trigger:** No `MSG_HTTP_REQUEST` has been received for more than 60 seconds while VS Code is running and a chat turn is active.  
**Rationale:** If the proxy is offline, CopilotScope cannot provide ground-truth token counts or price calculations. This may mean the proxy crashed or VS Code is not routing through the proxy.

---

## Adding Custom Rules

To add a rule to the anomaly engine (`src/daemon/anomaly_engine.cpp`):

```cpp
// In AnomalyEngine::RegisterRules(), add:
rules_.push_back({
    "CS-X999",                        // Unique rule ID
    "My Custom Rule",                 // Human-readable name
    AlertSeverity::SEVERITY_FLAG,     // Severity level
    [](const EventContext& ctx, const AnomalyEngine& eng) -> bool {
        // Return true to fire the alert
        if (ctx.msg_type != MSG_TOOL_CALL) { return false; }
        // Parse the JSON payload to check conditions
        // ...
        return /* your condition */;
    }
});
```

Rules receive the full `EventContext` (see `shared/event_types.h`) and have access to the engine's session state.

---

## Rule ID Namespace

| Prefix | Category |
|--------|---------|
| `CS-T` | Tool calls |
| `CS-C` | Context / compaction |
| `CS-M` | Model identity |
| `CS-N` | Network / API |
| `CS-S` | System / process |
| `CS-X` | Custom / user-defined |
