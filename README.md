# Edge Agent Runtime — Design Document
## Sarvam AI | Systems Design Assignment

---

## 1. Overview

Edge is a native desktop application for Windows 11 and macOS 14+ that runs Indic-language AI models on enterprise user devices. It exposes a local HTTP/SSE API consumed by enterprise web apps, Electron apps, and browser extensions.

**Core invariants:**

1. **Fault isolation** — a crash in the inference worker must never bring down the API server.
2. **Graceful degradation** — every accelerator failure transparently falls back to a slower-but-correct path without changing the public API contract.
3. **Multi-tenant fairness** — multiple MFEs sharing one local agent each get fair access to 4 concurrent inference slots.

**Recommended approach:** 5-process architecture (supervisor + API + active worker + standby worker + on-demand probe/updater) with hot-standby crash recovery, circuit-breaker-driven hardware fallback, and a shell-owned singleton SDK implementing weighted-fair-queuing.

---

## 2. System Architecture

```mermaid
graph TB
    subgraph "Enterprise Web App"
        subgraph "React Shell (Module Federation Host)"
            SDK["@edge/runtime SDK<br/>- HTTP keep-alive client<br/>- WFQ Scheduler<br/>- Auth token injection<br/>- AbortSignal propagation"]
            DocQA["Document Q&A MFE"]
            Meeting["Meeting Summariser MFE"]
        end
    end

    subgraph "Edge Agent (Native Binary)"
        SUP["edge-supervisor<br/>(persistent)<br/>- Spawn/monitor children<br/>- Write minidumps<br/>- Crash recovery<br/>- Trigger updates"]
        API["edge-api<br/>(persistent)<br/>- HTTP/SSE 127.0.0.1:7878<br/>- Auth & Validation<br/>- WFQ Scheduler<br/>- WAL writer<br/>- SSE encoder"]
        WA["edge-worker (active)<br/>(persistent)<br/>- ORT / Core ML<br/>- NPU/ANE/GPU/CPU<br/>- Circuit Breaker"]
        WS["edge-worker (standby)<br/>(persistent)<br/>- Pre-loaded weights<br/>- Idle, ready to promote"]
        PROBE["edge-driver-probe<br/>(on-demand)<br/>- NPU/ANE canary"]
        UPD["edge-updater<br/>(on-demand)<br/>- Download & verify"]
        MMAP["Shared Memory Region<br/>(mmap, read-only)<br/>- Model weights 4-8 GB<br/>- Tokeniser tables"]
    end

    DocQA -->|"agent.infer(...)"| SDK
    Meeting -->|"agent.infer(...)"| SDK
    SDK -->|"HTTP/1.1 + SSE<br/>127.0.0.1:7878<br/>Bearer token"| API

    SUP -->|spawns| API
    SUP -->|spawns| WA
    SUP -->|spawns| WS
    SUP -->|spawns| PROBE
    SUP -->|spawns| UPD

    API <-->|"IPC<br/>(named pipe / UDS)<br/>MessagePack frames"| WA
    WA ---|"mmap read-only"| MMAP
    WS ---|"mmap read-only"| MMAP
    PROBE -->|"probe_result IPC"| SUP
```

---

## 3. Process Inventory

| # | Process | Lifetime | Count | Responsibility |
|---|---|---|---|---|
| 1 | `edge-supervisor` | Persistent | 1 | Parent process. Spawns/monitors children, writes minidumps, applies updates, restores breaker state. |
| 2 | `edge-api` | Persistent | 1 | Local HTTP/1.1 + SSE server. Auth, validation, scheduler, SSE encoder, WAL writer. |
| 3 | `edge-worker` (active) | Persistent | 1 | Hosts ORT / Core ML. Inference execution against NPU/ANE/GPU/CPU. |
| 4 | `edge-worker` (standby) | Persistent | 1 | Hot spare. Pre-loaded with model weights via shared mmap. Promoted on crash. |
| 5 | `edge-driver-probe` | On-demand | 0–1 | Short-lived single-token canary inference for hardware health checks. |
| 6 | `edge-updater` | On-demand | 0–1 | Background download and signature verification of update packages. |

---

## 4. Process Lifecycle

### 4.1 Startup Sequence

```mermaid
sequenceDiagram
    participant OS
    participant SUP as edge-supervisor
    participant API as edge-api
    participant WA as edge-worker (active)
    participant WS as edge-worker (standby)

    OS->>SUP: Launch (T=0ms)
    SUP->>SUP: Read breakers.json (T=20ms)
    SUP->>SUP: mmap model weights (T=30ms)
    SUP->>API: Spawn (T=50ms)
    API->>API: Bind 127.0.0.1:7878
    API->>API: Open IPC server
    SUP->>WA: Spawn (T=80ms)
    WA->>WA: Inherit mmap weights
    WA->>WA: Init ORT sessions (QNN+CPU)
    WA-->>SUP: "ready" signal (T=400ms)
    SUP->>WS: Spawn (T=410ms)
    WS->>WS: Inherit mmap weights
    WS-->>SUP: "ready" signal (T=700ms)
    Note over API: Begin accepting HTTP requests
```

**Cold start time: ~700ms**

### 4.2 Shutdown Sequence

```mermaid
sequenceDiagram
    participant OS
    participant SUP as edge-supervisor
    participant API as edge-api
    participant W as workers

    OS->>SUP: SIGTERM / WM_QUERYENDSESSION (T=0)
    SUP->>API: IPC: shutdown (T=10ms)
    API->>API: Stop accepting connections (T=20ms)
    API->>API: Send error events to active streams (T=30ms)
    API->>API: Flush WAL with fsync (T=50ms)
    API->>W: IPC: shutdown (T=100ms)
    W->>W: Finish current token, emit request_end
    W-->>SUP: Exit cleanly (T=200ms)
    Note over SUP: Hard deadline at T+5000ms: force-kill remaining
```

---

## 5. Worker Crash Recovery

### 5.1 Recovery Sequence

```mermaid
sequenceDiagram
    participant Client as Client (Browser)
    participant API as edge-api
    participant SUP as edge-supervisor
    participant WA as edge-worker (active)
    participant WS as edge-worker (standby)

    Client->>API: GET /v1/infer (SSE open)
    API->>WA: IPC: request_start
    WA-->>API: token_chunk #0
    API-->>Client: data: chunk #0
    WA-->>API: token_chunk #1
    API-->>Client: data: chunk #1

    Note over WA: ✸ SIGSEGV / Access Violation

    WA--xSUP: Process death notification (T+0ms)
    API->>API: Detect IPC pipe close
    SUP->>SUP: Write minidump (T+10ms)
    SUP->>SUP: Promote standby → active (T+50ms)
    SUP-->>API: IPC: "promoted" (T+50ms)
    API->>WS: IPC: request_start (replay from WAL)
    API-->>Client: data: {"type":"recovery","delayMs":80}
    SUP->>SUP: Spawn new standby (T+80ms)
    WS-->>API: token_chunk #N (T+150ms)
    API-->>Client: data: chunk #N
    Note over Client: Stream resumes (~150ms stall)
    Note over SUP: New standby ready (T+500ms)
```

### 5.2 Why Warm Restart Beats Cold Start

| Factor | Cold Start | Warm Restart (Hot Standby) |
|---|---|---|
| Model weights loaded | 2–5 s disk read (4–8 GB) | 0 ms — already in OS page cache |
| ORT graph built | 500–1500 ms | 0 ms — done at standby spawn time |
| Driver/EP initialised | 200–800 ms | 0 ms — pre-initialised |
| Tokeniser loaded | 50–200 ms | 0 ms — in shared mmap |
| **Total user-visible stall** | **3–8 s** | **~150 ms** |

### 5.3 State Persisted to Disk

| File | Purpose | Format | Sync |
|---|---|---|---|
| `~/.edge/state/requests.wal` | In-flight request envelopes | Length-prefixed MessagePack | fsync every 100ms |
| `~/.edge/state/breakers.json` | Circuit breaker state + TTL | JSON | On every transition |
| `~/.edge/crashes/*.dmp` | Minidumps for postmortem | OS native format | On crash only |

---

## 6. Qualcomm NPU Failure Handling (Windows ARM64)

### 6.1 Failure Detection

The worker wraps every `OrtSession::Run()` call. The QNN execution provider surfaces driver errors as ORT statuses with a vendor HRESULT:

| ORT Status | QNN HRESULT | Classification | Action |
|---|---|---|---|
| OK | — | Success | Continue |
| FAIL | `ERROR_DEVICE_REMOVED` (0x800703E3) | **Unrecoverable** | Trip breaker immediately |
| FAIL | `ERROR_GEN_FAILURE` / timeout | Transient | Retry once, then trip breaker |
| FAIL | Other | Unknown | Log, retry on CPU |
| SEH exception | — | Process-fatal | Worker dies → supervisor handles |

### 6.2 CPU Fallback (No Restart Required)

The worker holds **two ORT sessions** throughout its lifetime:
- `session_npu` — configured with QNN Execution Provider
- `session_cpu` — configured with CPU Execution Provider

Both reference the same memory-mapped weights. Switching is a pointer flip:

```cpp
ORTSession* select_session() {
    return breaker.state() == CLOSED ? session_npu : session_cpu;
}
```

No process restart, no model reload, no API server interruption.

### 6.3 Circuit Breaker State Machine

```mermaid
stateDiagram-v2
    [*] --> CLOSED
    CLOSED --> OPEN : ERROR_DEVICE_REMOVED<br/>or 2 transient fails in 30s
    OPEN --> HALF_OPEN : Backoff timer elapsed<br/>(30s...30min, exponential)
    HALF_OPEN --> CLOSED : 5 consecutive canary successes
    HALF_OPEN --> OPEN : Canary fails<br/>(double backoff)

    state CLOSED {
        [*] --> RouteToNPU
        RouteToNPU : All requests → QNN_EP
    }
    state OPEN {
        [*] --> RouteToCPU
        RouteToCPU : All requests → CPU_EP
        RouteToCPU : NPU quarantined
    }
    state HALF_OPEN {
        [*] --> Probing
        Probing : edge-driver-probe runs canary
        Probing : 1-token inference on QNN_EP
    }
```

### 6.4 Latency Communication to Client

The backend choice is surfaced through three channels (API contract unchanged):

1. **HTTP response headers:**
   ```
   X-Edge-Backend: cpu
   X-Edge-Mode: degraded
   X-Edge-Latency-Class: batch
   ```

2. **SSE `stream_start` event:**
   ```json
   { "type": "stream_start", "backend": "cpu", "mode": "degraded", "estLatencyMs": 2400 }
   ```

3. **SSE `backend_change` event** (if backend changes mid-stream):
   ```json
   { "type": "backend_change", "from": "npu", "to": "cpu", "reason": "npu_unavailable" }
   ```

### 6.5 Quarantine Rules

- While `OPEN`: zero requests route to NPU. No per-request retry of the broken device.
- `edge-driver-probe` runs in a separate process (hung NPU driver cannot block the active worker).
- Breaker state persisted to `~/.edge/state/breakers.json` with 5-minute TTL.
- On agent restart: expired state discarded (reboot may have fixed driver); fresh state honoured.

---

## 7. Apple Neural Engine Fallback (macOS)

### 7.1 Fallback Decision Tree

```mermaid
flowchart TD
    A[Inference Request on macOS] --> B{Backend cache<br/>for this model?}
    B -->|Cache hit: ANE viable| C[Try ANE<br/>Core ML cpuAndNeuralEngine]
    B -->|Cache miss| C
    B -->|Cache says ANE banned| E[Try Metal GPU]

    C -->|Success| D[Return result<br/>X-Edge-Backend: ane]
    C -->|kCMErrorUnsupportedOperation| E2[Mark model as ANE-banned<br/>in persistent cache]
    E2 --> E

    E[Try Metal GPU<br/>Core ML cpuAndGPU / MPSGraph] -->|Success| F[Return result<br/>X-Edge-Backend: gpu]
    E -->|Failure| G[Try CPU<br/>Accelerate / BNNS / vDSP]

    G -->|Success AND<br/>latency within budget| H[Return result<br/>X-Edge-Backend: cpu]
    G -->|Failure OR<br/>p95 > threshold| I{Cloud eligible?<br/>1. User opt-in<br/>2. Online<br/>3. Model flagged OK}

    I -->|Yes| J[Try Cloud API<br/>X-Edge-Backend: cloud<br/>X-Edge-Privacy: data-left-device]
    I -->|No| K[Emit error:<br/>NO_BACKEND<br/>retryable: false]

    J -->|Success| L[Return result]
    J -->|Failure| K
```

### 7.2 Performance Trade-offs

| Backend | First-Token Latency (4B model) | Throughput | Power | Privacy |
|---|---|---|---|---|
| ANE (Core ML) | ~120 ms | ~80 tok/s | Lowest | On-device |
| Metal GPU | ~180 ms | ~60 tok/s | Medium | On-device |
| CPU (Accelerate) | ~600 ms | ~12 tok/s | Highest (battery drain) | On-device |
| Cloud (Sarvam hosted) | ~250 ms + RTT | Network-bound | Negligible local | **Off-device** (gated) |

### 7.3 Cloud Fallback Gates

Cloud is the last resort, gated by **three independent conditions** (all must be true):
1. User or enterprise policy has opted in
2. Device currently has internet connectivity
3. Model is flagged `cloud_eligible: true` (PII-handling models are never cloud-eligible)

### 7.4 API Contract Preservation

The HTTP request body, response status code, response shape, and SSE event types are **identical regardless of backend**. Backend selection is surfaced only through:
- Response headers: `X-Edge-Backend`, `X-Edge-Mode`, `X-Edge-Privacy`
- SSE events: `stream_start` (includes `backend` and `mode`), `backend_change`

A client that ignores these headers/events sees identical wire bytes whether it ran on ANE or in the cloud.

---

## 8. Micro-Frontend Integration Architecture

### 8.1 Decision: Shell-Owned Singleton SDK

```mermaid
graph TB
    subgraph "React Shell"
        SDK["@edge/runtime SDK (singleton)"]
        subgraph "MFEs (Module Federation remotes)"
            DQ["Document Q&A"]
            MS["Meeting Summariser"]
        end
    end

    DQ -->|"import { agent }<br/>from '@edge/runtime'"| SDK
    MS -->|"import { agent }<br/>from '@edge/runtime'"| SDK
    SDK -->|"Single keep-alive<br/>HTTP connection<br/>+ Bearer token"| AGENT["Edge Agent<br/>127.0.0.1:7878"]
```

### 8.2 Why Not Direct MFE-to-Localhost?

| Concern | Direct MFE-to-localhost | Shell-owned singleton |
|---|---|---|
| 4-slot fairness | Cannot enforce client-side | Central coordinator enforces caps |
| Auth token | Each MFE holds token (exposure risk) | Shell holds token; MFEs never see it |
| CORS | Agent must allow every MFE origin | Agent only allows Shell origin |
| Connection reuse | N connections for N MFEs | One keep-alive connection |
| Failure propagation | Crashed MFE leaks slot until timeout | Singleton tracks ownership, releases on unmount |
| API evolution | Every MFE updates independently | Shell ships new SDK; MFEs get it automatically |

### 8.3 Security Model

| Surface | Mitigation |
|---|---|
| Token theft from MFE | Token in SDK closure; no `agent.token` property |
| MFE bypasses scheduler | Agent rejects requests without `X-Edge-Sdk-Sig` header |
| CORS | Agent allowlist: Shell origin only |
| Token rotation | Shell fetches fresh token every 24h; SDK rotates seamlessly |
| DNS rebinding | Agent rejects `Host:` headers that aren't `localhost` / `127.0.0.1` |

---

## 9. Client-Side Request Scheduler

### 9.1 Architecture

```mermaid
graph TB
    subgraph "Shell SDK Scheduler"
        direction TB
        DQQ["MFE: doc-qa<br/>user queue: [r12, r15]<br/>bg queue: [r07]<br/>in-flight: 2"]
        MSQ["MFE: meeting-summariser<br/>user queue: []<br/>bg queue: [r19]<br/>in-flight: 1"]
        EXT["MFE: extension-foo<br/>user queue: [r20]<br/>bg queue: []<br/>in-flight: 1"]

        DISP["WFQ Dispatcher<br/>- Weighted round-robin across MFEs<br/>- User band > Background band<br/>- Per-MFE cap: 3/4<br/>- Aging: +1 priority / 5s"]
    end

    DQQ --> DISP
    MSQ --> DISP
    EXT --> DISP

    DISP --> S1["Slot 1"]
    DISP --> S2["Slot 2"]
    DISP --> S3["Slot 3"]
    DISP --> S4["Slot 4"]

    S1 --> AGENT["Edge Agent API"]
    S2 --> AGENT
    S3 --> AGENT
    S4 --> AGENT
```

### 9.2 Scheduling Policy

**Weighted Fair Queuing with Priority Bands and Aging:**

- **Two priority bands:** `user-triggered` (high) and `background-prefetch` (low)
- **Per-MFE soft cap:** 3 of 4 slots (at least 1 reserved for other MFEs)
- **Aging:** every 5s a background request waits, effective priority increments by 1
- **Promotion:** after 30s, background request is treated as user-priority
- **Dispatch order:** Deficit Round-Robin across MFEs; within an MFE, user band first

### 9.3 Concurrency Rules

| Property | Value |
|---|---|
| Global cap | 4 in-flight |
| Per-MFE soft cap | 3 in-flight |
| Per-MFE hard cap | 4 (only when no other MFE has pending work) |
| Slot grant | First-come-first-served within selected priority band |
| Slot release | On `stream_end`, `cancelled`, `timeout`, or `error` |

### 9.4 Backpressure Handling

- `agent.infer()` returns `AsyncIterable<InferEvent>` immediately (even if queued)
- First yielded event for a queued request: `{ type: 'queued', position, etaMs, reason }`
- If slot acquired within timeout: normal stream follows
- If queue timeout (60s) elapses: yields `{ type: 'timeout', at: 'queue' }` and ends

### 9.5 Cancellation Flow

```mermaid
flowchart TD
    A[MFE calls controller.abort] --> B{Request state?}
    B -->|QUEUED| C[Remove from queue<br/>Yield: cancelled at queue<br/>Close iterator]
    B -->|INFLIGHT| D[Send DELETE /v1/requests/id<br/>Wait for SSE cancelled event<br/>Yield: cancelled at inflight<br/>Close iterator]
    B -->|Already terminal| E[No-op - idempotent]
```

### 9.6 Scheduler State Machine (Per Request)

```mermaid
stateDiagram-v2
    [*] --> SUBMITTED
    SUBMITTED --> QUEUED : enqueue
    QUEUED --> ACQUIRING : slot reserved
    ACQUIRING --> INFLIGHT : first SSE event
    INFLIGHT --> COMPLETED : stream_end

    QUEUED --> TERMINATED : abort / timeout
    ACQUIRING --> TERMINATED : abort
    INFLIGHT --> TERMINATED : abort / timeout / error

    INFLIGHT --> RECOVERING : worker crash
    RECOVERING --> INFLIGHT : replay succeeds
    RECOVERING --> TERMINATED : replay fails (2x)
```

---

## 10. Streaming + Capacity Contention Scenario

### 10.1 Scenario

- **Slot 1:** Meeting Summariser streaming (request M-201)
- **Slots 2, 3, 4:** in use by other clients
- **Action:** User opens Document Q&A and submits a query

### 10.2 Sequence

```mermaid
sequenceDiagram
    participant DQ as Document Q&A
    participant SDK as @edge/runtime SDK
    participant API as edge-api
    participant MS as Meeting Summariser

    Note over API: All 4 slots occupied

    DQ->>SDK: agent.infer({...}, {priority: 'user'})
    SDK->>SDK: Scheduler: enqueue at user-priority
    SDK-->>DQ: yield {type:'queued', position:1, etaMs:4200}
    SDK-->>MS: yield {type:'capacity_changed', queuedBehindYou:1}

    Note over SDK: 4.2 seconds pass...

    API-->>SDK: Slot 3 freed (other request completed)
    SDK->>SDK: Scheduler: dispatch DQ request
    SDK-->>DQ: yield {type:'slot_acquired', waitedMs:4200}
    SDK->>API: POST /v1/infer
    API-->>SDK: SSE: stream_start
    SDK-->>DQ: yield {type:'stream_start', backend:'npu', mode:'optimal'}
    SDK-->>MS: yield {type:'capacity_changed', queuedBehindYou:0}

    loop Token generation
        API-->>SDK: SSE: chunk
        SDK-->>DQ: yield {type:'chunk', text:'...', index:N}
    end

    API-->>SDK: SSE: stream_end
    SDK-->>DQ: yield {type:'stream_end', tokens:312, durationMs:1790}
```

### 10.3 What Each MFE Receives

**Document Q&A (new request):**
```
T=0ms      → { type: 'queued', position: 1, etaMs: 4200, reason: 'capacity' }
T=200ms    → { type: 'position', position: 1, etaMs: 4000 }  (heartbeat)
T=4200ms   → { type: 'slot_acquired', waitedMs: 4200 }
T=4210ms   → { type: 'stream_start', backend: 'npu', mode: 'optimal', estLatencyMs: 1800 }
T=4250ms   → { type: 'chunk', text: 'The', index: 0 }
...
T=6000ms   → { type: 'stream_end', tokens: 312, durationMs: 1790, finishReason: 'stop' }
```

**Meeting Summariser (already streaming):**
```
T=5ms      → { type: 'capacity_changed', queuedBehindYou: 1 }
(continues normal chunk events uninterrupted)
T=4200ms   → { type: 'capacity_changed', queuedBehindYou: 0 }
```

### 10.4 Queue Visibility Rules

| What a queued request sees | What it does NOT see |
|---|---|
| ✅ Its own position | ❌ Other MFEs' identities |
| ✅ Its own ETA | ❌ Other request IDs |
| ✅ Its queue reason | ❌ Other models/prompts |

| What an active stream sees | |
|---|---|
| ✅ Count of requests waiting behind it | ❌ No identifying info |

### 10.5 Loading/Progress UX Semantics

| State | UI Hint | Visual Recommendation |
|---|---|---|
| `queued` | `uiHint: 'queue'` | Progress bar with position, "Waiting…" label |
| `inflight` | `uiHint: 'progress'` | Token-by-token rendering, blinking cursor |

### 10.6 Cancellation by State

| State | Round-trip? | Latency | Final Event |
|---|---|---|---|
| QUEUED | No | <5 ms | `{ type: 'cancelled', at: 'queue' }` |
| ACQUIRING | One IPC | <20 ms | `{ type: 'cancelled', at: 'queue' }` |
| INFLIGHT | HTTP DELETE + IPC | <100 ms | `{ type: 'cancelled', at: 'inflight' }` |
| Already terminal | None | 0 ms | No event (idempotent) |

### 10.7 Retry Semantics

| Trigger | Auto-retry? | Notes |
|---|---|---|
| `error, retryable: true` | No (MFE decides) | SDK exposes error; MFE can re-submit |
| `error, retryable: false` | No | Final |
| `timeout, at: 'queue'` | No | MFE can re-submit |
| Worker crash mid-stream | **Yes** (once, by agent) | Surfaced as `recovery` event |
| Network error to localhost | **Yes** (3 attempts, exp backoff) | Then `AGENT_UNREACHABLE` |

---

## 11. API Contract Examples

### 11.1 Inference Request

```http
POST /v1/infer HTTP/1.1
Host: 127.0.0.1:7878
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Content-Type: application/json
Accept: text/event-stream
X-Edge-Mfe-Id: doc-qa

{
  "model": "sarvam-indic-4b-instruct",
  "prompt": "Summarise this document in Hindi...",
  "params": {
    "maxTokens": 512,
    "temperature": 0.7,
    "topP": 0.95
  },
  "metadata": { "requestKind": "user" }
}
```

### 11.2 SSE Response (Degraded Mode)

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
X-Edge-Backend: cpu
X-Edge-Mode: degraded
X-Edge-Request-Id: r-7f3a2b9c

data: {"type":"queued","position":1,"etaMs":3800,"reason":"capacity"}

data: {"type":"slot_acquired","waitedMs":3850}

data: {"type":"stream_start","backend":"cpu","mode":"degraded","estLatencyMs":4500}

data: {"type":"chunk","text":"The","index":0}

data: {"type":"chunk","text":" document","index":1}

data: {"type":"stream_end","tokens":48,"durationMs":4321,"finishReason":"stop"}
```

### 11.3 Health Endpoint

```json
{
  "status": "ok",
  "subsystems": {
    "supervisor": "ok",
    "api": "ok",
    "worker_active": "ok",
    "worker_standby": "ok",
    "scheduler": { "inflight": 2, "queued": 0 },
    "circuit_breaker_npu": { "state": "closed", "failures": 0 }
  },
  "platform": "windows-arm64",
  "buildId": "1.4.2+a3b9c0d"
}
```


---

## 12. SSE Event Schema (TypeScript)

```typescript
export type Backend = 'npu' | 'ane' | 'gpu' | 'cpu' | 'cloud';
export type Mode = 'optimal' | 'degraded' | 'emergency';

export type ErrorCode =
  | 'NO_BACKEND'
  | 'MODEL_NOT_LOADED'
  | 'INVALID_PROMPT'
  | 'AUTH_FAILED'
  | 'AGENT_UNREACHABLE'
  | 'INTERNAL_ERROR'
  | 'SHUTTING_DOWN';

export type InferEvent =
  | { type: 'queued';           position: number; etaMs: number;
      reason: 'capacity' | 'quarantine'; uiHint: 'queue' }
  | { type: 'position';         position: number; etaMs: number;
      uiHint: 'queue' }
  | { type: 'slot_acquired';    waitedMs: number; uiHint: 'progress' }
  | { type: 'stream_start';    backend: Backend; mode: Mode;
      estLatencyMs: number; uiHint: 'progress' }
  | { type: 'chunk';           text: string; index: number }
  | { type: 'tool_call';      name: string; args: unknown }
  | { type: 'capacity_changed'; queuedBehindYou: number }
  | { type: 'recovery';       delayMs: number }
  | { type: 'backend_change'; from: Backend; to: Backend; reason: string }
  | { type: 'stream_end';     tokens: number; durationMs: number;
      finishReason: 'stop' | 'length' | 'tool' }
  | { type: 'cancelled';      at: 'queue' | 'inflight' }
  | { type: 'timeout';        at: 'queue' | 'inflight'; elapsedMs: number }
  | { type: 'error';          code: ErrorCode; message: string;
      retryable: boolean };
```

---

## 13. IPC Message Format

**Transport:** Named pipe (Windows) / Unix domain socket (macOS)

**Frame format:**
```
┌──────────────────┬────────────────────────────────────┐
│ length (u32 BE)  │  MessagePack-encoded payload       │
│ 4 bytes          │  variable length                   │
└──────────────────┴────────────────────────────────────┘
```

**Message types:**
```typescript
type IpcMessage =
  | { kind: 'request_start'; id: string; model: string; prompt: string;
      params: { maxTokens: number; temperature: number; topP: number } }
  | { kind: 'token_chunk';   id: string; index: number; text: string }
  | { kind: 'request_end';   id: string; tokens: number; durationMs: number;
      finishReason: 'stop' | 'length' | 'cancelled' | 'error' }
  | { kind: 'cancel';        id: string }
  | { kind: 'control';       op: 'probe_result' | 'breaker_state' | 'shutdown';
      payload: unknown };
```

---

## 14. Data Models

### 14.1 WAL Record

```typescript
interface WalRecord {
  request_id: string;       // ULID
  mfe_id: string;
  model: string;
  prompt: string;
  params: { maxTokens: number; temperature: number; topP: number };
  priority: 'user' | 'background';
  enqueued_at_ms: number;
  state: 'queued' | 'acquiring' | 'inflight' | 'done' | 'failed';
  worker_pid: number;       // 0 if not yet dispatched
  last_token_index: number; // -1 if no tokens yet
}
```

### 14.2 Circuit Breaker State (`breakers.json`)

```json
{
  "version": 1,
  "written_at_ms": 1715000000000,
  "ttl_ms": 300000,
  "breakers": {
    "qnn_ep": {
      "state": "open",
      "open_since_ms": 1714999700000,
      "next_probe_at_ms": 1715000060000,
      "consecutive_failures": 1,
      "current_backoff_ms": 60000
    },
    "ane": {
      "state": "closed",
      "consecutive_failures": 0
    }
  }
}
```

---

## 15. Error Handling

### 15.1 Error Classification

| Class | Examples | Retryable | User-visible |
|---|---|---|---|
| Transient hardware | QNN timeout | Yes (after fallback) | `degraded` mode notice |
| Unrecoverable hardware | `ERROR_DEVICE_REMOVED` | Yes (via fallback chain) | `backend_change` event |
| Worker crash | SIGSEGV | Yes (via standby) | `recovery` event |
| Resource exhaustion | OOM during model load | No | `error: NO_BACKEND` |
| Validation | Invalid model, prompt too long | No | HTTP 422 |
| Auth | Missing/invalid token | No | HTTP 401 |
| Capacity | Queue timeout (60s) | No (MFE re-submits) | `timeout, at: 'queue'` |

### 15.2 Error Propagation

```mermaid
flowchart LR
    A[Worker crash] --> B[Supervisor detects]
    B --> C[Standby promoted]
    C --> D["SSE: recovery event"]
    D --> E[Stream resumes]

    F[Driver hard error] --> G[Breaker opens]
    G --> H[CPU fallback]
    H --> I["SSE: backend_change"]

    J[Network error] --> K[SDK retries 3x]
    K --> L{Success?}
    L -->|Yes| M[Continue]
    L -->|No| N["error: AGENT_UNREACHABLE"]
```

---

## 16. Performance and Reliability

### 16.1 Performance Targets

| Metric | NPU/ANE (optimal) | CPU (degraded) |
|---|---|---|
| Time-to-first-token (4B model) | < 250 ms | < 1500 ms |
| Throughput (tokens/sec) | ≥ 60 | ≥ 8 |
| API dispatch overhead | < 5 ms | < 5 ms |
| SSE latency per token | < 2 ms | < 2 ms |
| Cold start | < 1 s | < 1 s |
| Warm restart after crash | < 200 ms | < 500 ms |

### 16.2 Memory Budget

| Component | RSS |
|---|---|
| `edge-supervisor` | < 50 MB |
| `edge-api` | < 80 MB |
| `edge-worker` (active) | model_size + 200 MB |
| `edge-worker` (standby) | 100 MB (activation buffers only) |
| Shared mmap region | model_size (4–8 GB) |

### 16.3 Reliability

| Failure | MTTR | Mitigation |
|---|---|---|
| Worker crash | < 200 ms | Hot standby + WAL replay |
| API server crash | < 1 s | Supervisor respawns; SDK retries |
| NPU driver failure | < 100 ms | Circuit breaker; CPU fallback |
| Disk full | Immediate | Continue without WAL; replay disabled |
| Agent fully dead | SDK error | MFE shows reconnect UX; OS auto-restart |

---

## 17. Design Trade-off Summary

| Decision | Alternative | Why This | What We Give Up |
|---|---|---|---|
| 5-process tree | Monolithic | Fault isolation (required) | More IPC, more memory |
| Hot-standby worker | Cold restart | Sub-second recovery | ~100 MB memory |
| Shared mmap weights | Per-process load | Standby stays cheap | Platform-specific code |
| Circuit breaker | Eager fallback | Avoids losing NPU on transient blips | More code |
| Shell-owned SDK | Direct MFE-to-localhost | Central auth, fairness, telemetry | Version coupling |
| WFQ + aging | Strict priority | Prevents starvation | More complex |
| SSE streaming | WebSocket | Simpler, HTTP-compatible | One-way only |
| Bearer token | Cookie | No CSRF; works in extensions | Must pass every call |
| 127.0.0.1 only | Unix socket | Works in browsers | Need DNS-rebinding defence |
