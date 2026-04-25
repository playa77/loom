# DOCUMENT 1: DESIGN DOCUMENT

## 1. OVERVIEW & GOALS

### 1.1 Core Purpose
A local, Ubuntu 24.04+ desktop application that provides a high-performance, OpenRouter-exclusive LLM chat interface. The system routes all inference requests through OpenRouter's API, supports local Python plugin execution, maintains complete conversation history without automatic truncation, and guarantees zero UI performance degradation at 250,000+ token contexts. The architecture enforces a strict, explicit separation between the **Context Window** (the `messages` array containing system prompt, conversation history, memories, knowledge, and attachments) and **Runtime Parameters** (model selection, temperature, top_p, thinking/reasoning toggle, tool definitions, search configuration, and other generation settings). Context window mutation is restricted to exactly six discrete UI-triggered events. Runtime parameters are fully dynamic and reactive.

### 1.2 Concrete, Verifiable Goals
- **Strict Context Window Immutability:** The `messages` array transmitted to OpenRouter is strictly immutable except during exactly six discrete UI-triggered events: (1) User submits a message, (2) LLM completes a generation, (3) UI attaches personal memories, (4) UI attaches project knowledge, (5) User uploads an attachment, (6) User modifies system instructions mid-conversation. Zero automatic pruning, summarization, sliding windows, background edits, or silent context manipulation occurs under any circumstances.
- **In-Memory Payload Assembly & Flat System Prompt:** The live conversation payload is assembled 100% in-memory within the Rust frontend's reactive state. SQLite is used exclusively for persistence, crash recovery, and historical audit. The `messages` array always contains exactly one `role: "system"` message at index 0. Mid-conversation system prompt overrides atomically replace `messages[0].content` in-memory. No SQL reconstruction, no hierarchical prompt merging, no template injection at request time.
- **Explicit Cache Invalidation Acceptance:** Changing `messages[0]` will invalidate provider-side prompt caching (OpenRouter, Anthropic, OpenAI, etc.). This is an inherent, accepted constraint of the API specification. The system will not attempt to preserve, bypass, or mitigate cache misses. Cache metadata headers returned by OpenRouter are logged for telemetry but ignored for routing or retry logic.
- **Fully Dynamic Runtime Parameters:** Model selection, temperature, top_p, top_k, presence/frequency penalties, reasoning/thinking level, tool definitions, and search toggles are fully decoupled from the context window. They can be modified at any point during an active conversation. Changes are applied atomically to the next generation request with zero latency, zero UI friction, and zero context reload. Parameter state is reactive and instantly synchronized across the UI and outbound payload.
- **250k+ Token UI Performance:** The Rust frontend renders 100% of conversation history using GPU-accelerated virtualization with chunked DOM recycling. Frame rate remains ≥60 FPS during continuous scrolling, anchor-based navigation, streaming, and collapsible block expansion at maximum tested context size.
- **OpenRouter-Exclusive Routing:** All inference requests target OpenRouter's `/v1/chat/completions` endpoint. The system natively handles OpenRouter-specific response formats, including `reasoning`/`thought` blocks, tool call payloads, provider metadata, and exact usage headers.
- **Explicit Error Recovery:** On OpenRouter 4xx/5xx errors, mid-stream drops, or context limit violations, streaming pauses immediately. The UI surfaces a non-dismissable prompt requiring manual model selection or retry confirmation before resuming. Zero silent fallbacks.
- **Plugin Isolation & Stability:** Local Python plugins execute as isolated subprocesses wrapped in `cgroups v2` scopes. Memory, CPU, and timeout limits are enforced at the kernel level. Plugin crashes, OOM events, or infinite loops never propagate to the backend or frontend.
- **Cross-Distro Secret Security:** API keys are stored using a tiered fallback chain: primary `libsecret`/Secret Service API, fallback AES-256-GCM encrypted file with machine-ID-derived key. Zero plaintext storage on disk across all Ubuntu-based distributions.

### 1.3 Explicit Non-Goals
- No automatic context summarization, sliding windows, token pruning, or background payload mutation.
- No SQL-based conversation reconstruction at request time. All payload assembly occurs in-memory.
- No hierarchical prompt templates, dynamic system prompt injection, or multi-layered context merging.
- No coupling of runtime parameter changes to context window state. Parameter updates never trigger context reloads or message array modifications.
- No cloud synchronization, multi-device state sharing, or remote backend hosting.
- No support for non-OpenRouter inference providers or alternative API formats.
- No in-process Python execution, FFI embedding, or shared event loops between Rust and Python.
- No silent fallback model switching, automatic retries, or hidden context limit workarounds.
- No partial or lazy-loaded conversation rendering. The UI always holds and renders the complete dataset.
- No prompt cache preservation, bypass, or mitigation strategies. Cache invalidation on system prompt override is explicitly accepted.

---

## 2. SYSTEM ARCHITECTURE

### 2.1 Component Breakdown
The system operates as a process-separated desktop application with three primary runtime components:

1. **Rust Frontend (Tauri)**
   - Maintains two strictly decoupled state domains:
     - **Context Window State:** Immutable `messages` array stored in a reactive `Signal<Vec<Message>>`. Modified exclusively via the six permitted triggers.
     - **Runtime Parameter State:** Reactive configuration object (`model`, `temperature`, `top_p`, `thinking_level`, `tools`, `search_enabled`, etc.) stored in `Signal<RuntimeConfig>`. Fully dynamic, updated instantly on user interaction.
   - **In-Memory Payload Assembly:** At generation time, the frontend merges the live `messages` array with the live `RuntimeConfig` into a single JSON payload. No database queries, no SQL joins, no template rendering.
   - **System Prompt Override Mechanism:** When the user edits the system prompt, the frontend atomically replaces `messages[0].content` in-memory. This triggers Context Trigger #6, fires the `tiktoken` refresh, updates the local context meter, and marks the payload as "dirty" for the next generation request. The UI shows zero loading state, zero scroll jump, and zero history reload.
   - Handles UI rendering, virtualized list management, dual-scroll mechanics, and collapsible block parsing.
   - Communicates with the Python backend via HTTP/WebSocket over `127.0.0.1:8080` or Unix Domain Socket.

2. **Python Backend (FastAPI + Uvicorn)**
   - Acts as a strict, stateless API proxy and orchestration layer.
   - Receives exact `messages` array and exact `parameters` object from the frontend. Forwards both verbatim to OpenRouter.
   - Manages OpenRouter HTTP sessions, SSE streaming, retry logic, and fallback prompting.
   - Spawns isolated plugin subprocesses with `cgroups v2` enforcement.
   - Handles SQLite persistence, secret retrieval, telemetry logging, and conversation state versioning.
   - Never modifies, truncates, or reorders the `messages` array. Never alters runtime parameters. Never reconstructs payloads from SQL.

3. **SQLite Persistence Layer**
   - Stores conversations, messages, attachments, personal memories, project knowledge, system prompt versions, runtime parameter snapshots, plugin configurations, usage metrics, and application settings.
   - Operates in WAL mode with synchronous commits for crash resilience.
   - Indexed for fast anchor-based navigation, context assembly, and parameter retrieval.
   - Used exclusively for historical audit, crash recovery, and UI state restoration on application launch. Not used for real-time payload generation.

### 2.2 Data Flow & IPC Contract
- **Generation Flow:** Rust UI merges live `messages` (context window) + live `parameters` (runtime config) → Serializes to JSON → POSTs to `/api/v1/chat/stream` → Python validates schema → Forwards both objects verbatim to OpenRouter → Streams SSE tokens back to Rust → Rust renders incrementally.
- **System Prompt Override Flow:** User edits system prompt → Rust validates input → Atomically replaces `messages[0].content` in-memory → Fires Trigger #6 → Calls `tiktoken.refresh()` → Sends `PATCH /api/v1/conversations/{id}/system-prompt` to Python → Python persists new version to SQLite → Returns `200 OK` → Next generation request automatically includes updated `messages[0]`. Zero latency impact, zero SQL reconstruction.
- **Runtime Parameter Update Flow:** User changes temperature/model/thinking level → Rust updates reactive parameter state → Persists to SQLite → Returns `200 OK` → Next generation request automatically includes updated parameters. Zero context reload, zero UI interruption.
- **Error Flow:** OpenRouter returns error/drop → Python catches exception → Halts SSE → Returns structured error payload → Rust pauses UI → Displays manual recovery prompt → User selects fallback model → Rust resubmits exact payload to new model endpoint.
- **Plugin Flow:** Rust detects tool call → Sends tool request to `/api/v1/plugins/execute` → Python spawns subprocess in `cgroups` scope → Captures JSON stdout/stderr → Returns structured result → Rust injects into conversation context (trigger #2 or #5).

### 2.3 Architecture Diagram (Mermaid)
```mermaid
graph TD
    subgraph "Rust Frontend (Tauri)"
        UI[Virtualized Chat Renderer]
        CW[Context Window State (In-Memory)]
        RP[Runtime Parameter State (In-Memory)]
        PA[Payload Assembler (In-Memory)]
        TC[tiktoken Counter]
        DS[Dual Scroll Controller]
        CB[Collapsible Block Manager]
        SPM[System Prompt Manager]
    end

    subgraph "Python Backend (FastAPI)"
        API[HTTP/SSE Router]
        OR[OpenRouter Proxy]
        PL[Plugin Orchestrator]
        DB[SQLite Manager (Persistence Only)]
        SEC[Secret Manager]
    end

    subgraph "External / OS"
        OPEN[OpenRouter API]
        CG[cgroups v2 Sandbox]
        KEY[libsecret / Encrypted Vault]
    end

    CW -->|Immutable Messages| PA
    RP -->|Dynamic Parameters| PA
    SPM -->|Atomic Index-0 Replacement| CW
    PA -->|Exact Payload| API
    API -->|Forward Request| OR
    OR -->|SSE Stream| API
    API -->|Token Chunks| UI
    UI -->|Render| DS
    UI -->|Render| CB
    TC -->|Refresh on 6 Triggers| CW
    API -->|Execute Plugin| PL
    PL -->|Spawn Subprocess| CG
    CG -->|JSON Output| PL
    PL -->|Result| API
    API -->|Persist State| DB
    SEC -->|Fetch Key| OR
    KEY -->|Decrypt/Retrieve| SEC
```

---

## 3. DATA MODEL

### 3.1 Conceptual Entities & Relationships
- **Conversation:** Root entity containing metadata, creation timestamp, linked project ID, and foreign keys to active system prompt version and runtime parameter snapshot.
- **Message:** Immutable record of a single turn. Contains `role` (user/assistant/system/tool), `content`, `timestamp`, `token_count`, and `metadata` (reasoning blocks, tool calls, attachment references). Forms the core of the Context Window. Stored in SQLite for persistence only.
- **SystemPromptVersion:** Versioned record of system instruction overrides. Contains `conversation_id`, `version_number`, `content`, `created_at`, and `is_active`. Enables atomic switching, historical auditability, and crash recovery. Not used for real-time payload assembly.
- **RuntimeParameters:** JSON-serialized configuration object storing `model`, `temperature`, `top_p`, `top_k`, `presence_penalty`, `frequency_penalty`, `thinking_level`, `tool_definitions`, `search_enabled`, and `max_tokens`. Updated dynamically, persisted per conversation.
- **Attachment:** Stores file metadata, MIME type, size, local path, and extracted text/chunks. Linked to specific messages or conversation context.
- **PersonalMemory / ProjectKnowledge:** Structured knowledge bases. Linked to conversations via explicit attachment events. Never auto-injected; only applied during the designated context modification trigger.
- **PluginRegistry:** Stores plugin metadata, execution parameters, timeout/memory limits, and sandbox configuration.
- **UsageMetric:** Tracks token counts, estimated costs, model usage, and generation timestamps per conversation. Updated exactly on the six permitted triggers.

### 3.2 Persistence Layer Strategy
- **Engine:** SQLite 3.45+ with WAL mode enabled.
- **Synchronization:** `PRAGMA synchronous = FULL` for crash resilience. `PRAGMA journal_mode = WAL` for concurrent read/write performance.
- **Indexing:** Composite indexes on `(conversation_id, created_at)` for message retrieval, `(conversation_id, is_active)` for system prompt resolution, and `(conversation_id, token_count)` for anchor navigation.
- **Migration Strategy:** Sequential, versioned SQL migrations. Zero destructive schema changes. Backward-compatible column additions only.
- **State Restoration:** On application launch, the Rust frontend loads the latest `messages` and `system_prompt_versions` from SQLite into in-memory `Signal` state. All subsequent operations occur in-memory. SQLite is never queried during active generation.

---

## 4. API / INTERFACE DESIGN

### 4.1 External Interfaces & Protocols
- **Rust ↔ Python IPC:** HTTP/1.1 over `127.0.0.1:8080` (fallback to Unix Domain Socket `/tmp/openrouter_client.sock`). JSON payloads for synchronous requests. Server-Sent Events (SSE) for streaming token delivery.
- **Python ↔ OpenRouter:** HTTPS to `https://openrouter.ai/api/v1`. Standard OpenAI-compatible `/chat/completions` endpoint. Bearer token authentication. SSE streaming enabled by default.
- **Plugin Execution:** Local Python subprocess invocation via `subprocess.Popen` with `systemd-run` cgroup wrapping. JSON-serialized stdin/stdout/stderr.

### 4.2 Core Endpoints
- `POST /api/v1/chat/stream`
  - **Request:**
    ```json
    {
      "conversation_id": "uuid",
      "messages": [
        {"role": "system", "content": "current_system_prompt"},
        {"role": "user", "content": "..."},
        {"role": "assistant", "content": "..."}
      ],
      "parameters": {
        "model": "string",
        "temperature": 0.7,
        "top_p": 1.0,
        "thinking_level": "high",
        "tools": [...],
        "search_enabled": false
      }
    }
    ```
  - **Response:** SSE stream with `data: {"token": "...", "type": "text|reasoning|tool_call"}` chunks. Terminates with `data: {"done": true, "usage": {...}}`.
- `PATCH /api/v1/conversations/{id}/system-prompt`
  - **Request:** `{ "content": "string", "version": int }`
  - **Response:** `{ "status": "applied", "new_version": int, "payload_hash": "sha256" }`
  - **Behavior:** Atomic SQLite update. Returns immediately. Frontend applies to next generation request. Zero latency impact on active UI.
- `PATCH /api/v1/conversations/{id}/parameters`
  - **Request:** `{ "parameters": { "temperature": 0.9, "model": "anthropic/claude-3.5-sonnet", ... } }`
  - **Response:** `{ "status": "updated", "applied_at": "timestamp" }`
  - **Behavior:** Instant state sync. No context reload. Applied to next generation request.
- `POST /api/v1/plugins/execute`
  - **Request:** `{ "plugin_id": "string", "params": {...}, "timeout_sec": 30, "memory_mb": 512 }`
  - **Response:** `{ "status": "success|error", "result": {...}, "stderr": "string", "execution_time_ms": int }`

### 4.3 Authentication & Session Management
- **Local IPC Trust:** No authentication required for `127.0.0.1` or Unix socket. Process isolation and OS-level permissions enforce boundary.
- **OpenRouter Auth:** Bearer token retrieved from `keyring` or encrypted vault at backend startup. Cached in memory. Never logged or persisted.
- **Session State:** Stateless backend. All conversation state, system prompt versions, runtime parameters, and context payloads are explicitly transmitted per request.

---

## 5. SECURITY & INFRASTRUCTURE

### 5.1 Secret Management Fallback Chain
1. **Primary:** `keyring` library utilizing Secret Service API (`libsecret`). Compatible with GNOME Keyring, KDE Wallet, and freedesktop.org implementations.
2. **Fallback:** AES-256-GCM encrypted file at `~/.config/openrouter_client/.secrets.enc`. Permissions: `0600`. Encryption key derived via HKDF-SHA256 from `/etc/machine-id` + application-specific salt.
3. **Runtime:** Key loaded into memory at startup. Zero disk writes post-initialization. Cleared on process exit.

### 5.2 Deployment Targets & Resource Requirements
- **OS:** Ubuntu 24.04 LTS (kernel 6.8+). Compatible with all Ubuntu-based derivatives (Kubuntu, Lubuntu, Pop!_OS, Bodhi, etc.).
- **Runtime Dependencies:** Python 3.12+, Rust 1.75+, Tauri 2.x, SQLite 3.45+, systemd (for cgroups v2).
- **Memory Footprint:** Frontend: ≤150MB idle, ≤400MB at 250k tokens. Backend: ≤80MB idle, ≤250MB during active streaming/plugin execution.
- **CPU:** Single-core baseline. Multi-core utilized only during parallel plugin execution and `tiktoken` counting.

### 5.3 Plugin Sandboxing & Isolation
- **Execution Wrapper:** `systemd-run --scope -p MemoryMax=512M -p CPUQuota=100% -p TimeoutStopSec=30 -p ProtectSystem=strict -p ReadWritePaths=~/.config/openrouter_client/plugins`
- **Process Tree Enforcement:** `cgroups v2` limits apply to all child processes spawned by the plugin. Prevents fork bombs, memory leaks, and resource exhaustion.
- **Network/Filesystem:** Network access permitted for web search/tools. Filesystem restricted to plugin directory + read-only system paths. No access to user home or sensitive configs.

---

## 6. KEY DESIGN DECISIONS

### 6.1 Strict Separation: Context Window vs. Runtime Parameters
**Rationale:** Conflating conversation history (`messages`) with generation configuration (`temperature`, `model`, `tools`, `thinking_level`) creates state coupling that causes UI jank, unnecessary context reloads, and unpredictable behavior during mid-conversation tuning. By enforcing strict immutability on the context window (except for six explicit triggers) while keeping runtime parameters fully dynamic and reactive, the system guarantees deterministic context state, zero-friction parameter adjustment, and atomic payload assembly at generation time.

### 6.2 In-Memory Payload Assembly & Flat System Prompt at Index 0
**Rationale:** Reconstructing the conversation payload from SQL at request time introduces latency, race conditions, and complex join logic. By maintaining the live `messages` array entirely in-memory within the Rust frontend's reactive state, the system guarantees zero-latency payload assembly. The system prompt is strictly positioned at `messages[0]`. Mid-conversation overrides atomically replace `messages[0].content` in-memory. SQLite is used exclusively for persistence and crash recovery. This eliminates SQL reconstruction overhead and ensures exact payload fidelity.

### 6.3 Explicit Cache Invalidation Acceptance
**Rationale:** Provider-side prompt caching relies on exact prefix matching. Changing `messages[0]` inherently breaks cache continuity. Attempting to preserve cache via prompt versioning, delta injection, or hierarchical merging introduces API incompatibility, unpredictable routing behavior, and silent state corruption. The system explicitly accepts cache invalidation as a direct, expected consequence of system prompt overrides. Cache metadata headers are logged for telemetry but ignored for routing, retry, or cost estimation logic.

### 6.4 Six-Trigger Context Window Immutability
**Rationale:** Automatic context manipulation (summarization, sliding windows, pruning) introduces silent data loss, hallucination amplification, and unpredictable model behavior. By restricting context window modification to exactly six explicit events, the system guarantees deterministic context state, full user control, and transparent API interactions. The UI maintains complete history; the backend transmits exact payloads.

### 6.5 Zero-Friction System Instruction Override
**Rationale:** Mid-conversation system prompt changes are critical for iterative prompting, role-switching, and constraint adjustment. Implementing this as an atomic, in-memory replacement with immediate frontend state sync ensures zero UI jank, zero context reload, and identical latency to model/temperature changes. The override replaces the `system` role message in the context window without altering conversation history or triggering SQL operations.

### 6.6 GPU-Accelerated Virtualized Rendering
**Rationale:** Rendering 250k+ tokens in a standard DOM causes layout thrashing, memory bloat, and frame drops. Chunked DOM recycling, virtualized list rendering, and GPU-composited scroll containers guarantee ≥60 FPS regardless of context size. Dual scrollbars (continuous + anchor-based) operate on virtualized indices, not raw DOM nodes.

### 6.7 Explicit Error Recovery & Manual Fallback
**Rationale:** Silent retries or automatic model switching mask API failures, corrupt conversation state, and violate user trust. Pausing streaming, surfacing exact error codes, and requiring manual model selection ensures transparency, preserves context integrity, and aligns with professional LLM workflow standards.

### 6.8 Cross-Distro Secret Fallback Chain
**Rationale:** Ubuntu derivatives use varying desktop environments and secret service implementations. A tiered fallback chain guarantees zero-plaintext storage across all configurations without requiring user configuration or breaking on DE swaps. Machine-ID-derived encryption keys provide deterministic, secure fallback without password prompts.

---
