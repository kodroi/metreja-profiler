# Metreja NDJSON Output Reference

Metreja writes **Newline-Delimited JSON** — one event per line. The first event is always `session_metadata`, followed by events matching the enabled event types configured via `set events`.

## Event Categories

| Category | Event Types | Emission Timing | maxEvents behavior |
|----------|-------------|----------------|--------------------|
| Metadata | `session_metadata` | First line, always emitted | Bypass |
| Per-call | `enter`, `leave`, `exception` | Real-time, one per method call/exit/throw | Count against cap |
| Aggregated | `method_stats`, `exception_stats` | At profiler shutdown | Bypass |
| Memory | `gc_start`, `gc_end` | Real-time, on GC events | Bypass |
| Memory | `alloc_by_class` | Real-time, per-type allocation | Count against cap |
| Contention | `contention_start`, `contention_end` | Real-time, via EventPipe (.NET 5+) | Count against cap |

## Event Type Schemas

### `session_metadata` (first event in file)

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Always `"session_metadata"` |
| `tsNs` | long | Timestamp in nanoseconds |
| `pid` | int | Process ID |
| `sessionId` | string | Session identifier (6-char hex) |
| `scenario` | string | Scenario name from session config |

```json
{"event":"session_metadata","tsNs":123456789,"pid":1234,"sessionId":"a1b2c3","scenario":"perf-test"}
```

### `enter` (method entry)

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Always `"enter"` |
| `tsNs` | long | Monotonic nanosecond timestamp (QPC-based) |
| `pid` | int | Process ID |
| `sessionId` | string | Session identifier |
| `tid` | int | OS thread ID |
| `depth` | int | Call stack depth (0 = top-level) |
| `asm` | string | Assembly name |
| `ns` | string | Namespace |
| `cls` | string | Class name |
| `m` | string | Method name (resolved for async) |
| `async` | bool | `true` if async state machine MoveNext |

```json
{"event":"enter","tsNs":123456789,"pid":1234,"sessionId":"a1b2c3","tid":5678,"depth":2,"asm":"MyApp","ns":"MyApp.Services","cls":"OrderService","m":"ProcessOrder","async":false}
```

### `leave` (method exit)

Same fields as `enter` plus:

| Field | Type | Description |
|-------|------|-------------|
| `deltaNs` | long | Elapsed nanoseconds from enter to leave |
| `tailcall` | bool | *(optional)* `true` if method exited via tail call |
| `wallTimeNs` | long | *(optional, async only)* Wall-clock time from first enter to final leave of the async method |

```json
{"event":"leave","tsNs":123556789,"pid":1234,"sessionId":"a1b2c3","tid":5678,"depth":2,"asm":"MyApp","ns":"MyApp.Services","cls":"OrderService","m":"ProcessOrder","async":false,"deltaNs":100000}
```

For async methods with wall-time tracking:
```json
{"event":"leave","tsNs":200,"pid":1234,"sessionId":"a1b2c3","tid":5,"depth":1,"asm":"MyApp","ns":"MyApp","cls":"Service","m":"DoWorkAsync","deltaNs":50,"async":true,"wallTimeNs":150000}
```

### `exception` (exception thrown)

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Always `"exception"` |
| `tsNs` | long | Timestamp in nanoseconds |
| `pid` | int | Process ID |
| `sessionId` | string | Session identifier |
| `tid` | int | OS thread ID |
| `asm` | string | Assembly name |
| `ns` | string | Namespace |
| `cls` | string | Class name |
| `m` | string | Method name |
| `exType` | string | Exception type (e.g., `System.NullReferenceException`) |

Note: exception events do **not** have `depth` or `async` fields.

```json
{"event":"exception","tsNs":123656789,"pid":1234,"sessionId":"a1b2c3","tid":5678,"asm":"MyApp","ns":"MyApp.Services","cls":"OrderService","m":"ProcessOrder","exType":"System.InvalidOperationException"}
```

### `method_stats` (aggregated method statistics)

Emitted at profiler shutdown. One event per unique method. The profiler tracks enter/leave timing in-process and aggregates call counts and timing without emitting per-call events.

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Always `"method_stats"` |
| `tsNs` | long | Always `0` (emitted at shutdown, not tied to a point in time) |
| `pid` | int | Process ID |
| `sessionId` | string | Session identifier |
| `asm` | string | Assembly name |
| `ns` | string | Namespace |
| `cls` | string | Class name |
| `m` | string | Method name (resolved for async) |
| `callCount` | long | Total number of times the method was called |
| `totalSelfNs` | long | Sum of self-time across all calls (excludes child method time) |
| `maxSelfNs` | long | Maximum self-time of any single call |
| `totalInclusiveNs` | long | Sum of inclusive-time across all calls (includes child method time) |
| `maxInclusiveNs` | long | Maximum inclusive-time of any single call |

```json
{"event":"method_stats","tsNs":0,"pid":1234,"sessionId":"a1b2c3","asm":"MyApp","ns":"MyApp.Services","cls":"OrderService","m":"ProcessOrder","callCount":150,"totalSelfNs":45000000,"maxSelfNs":1200000,"totalInclusiveNs":890000000,"maxInclusiveNs":12000000}
```

### `exception_stats` (aggregated exception statistics)

Emitted at profiler shutdown. One event per unique exception-type + method combination.

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Always `"exception_stats"` |
| `tsNs` | long | Always `0` (emitted at shutdown) |
| `pid` | int | Process ID |
| `sessionId` | string | Session identifier |
| `exType` | string | Exception type (e.g., `System.InvalidOperationException`) |
| `asm` | string | Assembly name of the throwing method |
| `ns` | string | Namespace of the throwing method |
| `cls` | string | Class name of the throwing method |
| `m` | string | Method name where the exception was thrown |
| `count` | long | Number of times this exception was thrown from this method |

```json
{"event":"exception_stats","tsNs":0,"pid":1234,"sessionId":"a1b2c3","exType":"System.InvalidOperationException","asm":"MyApp","ns":"MyApp.Services","cls":"OrderService","m":"ProcessOrder","count":12}
```

### `gc_start` (GC collection started)

Emitted when the corresponding event type is enabled via `set events`.

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Always `"gc_start"` |
| `tsNs` | long | Timestamp in nanoseconds |
| `pid` | int | Process ID |
| `sessionId` | string | Session identifier |
| `gen0` | bool | Generation 0 collected |
| `gen1` | bool | Generation 1 collected |
| `gen2` | bool | Generation 2 collected |
| `reason` | string | GC reason: `"induced"`, `"other"`, or `"unknown"` |

```json
{"event":"gc_start","tsNs":123456789,"pid":1234,"sessionId":"a1b2c3","gen0":true,"gen1":false,"gen2":false,"reason":"other"}
```

### `gc_end` (GC collection finished)

Emitted when the corresponding event type is enabled via `set events`.

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Always `"gc_end"` |
| `tsNs` | long | Timestamp in nanoseconds |
| `pid` | int | Process ID |
| `sessionId` | string | Session identifier |
| `durationNs` | long | GC pause duration in nanoseconds |

```json
{"event":"gc_end","tsNs":123556789,"pid":1234,"sessionId":"a1b2c3","durationNs":1500000}
```

### `alloc_by_class` (per-type allocation summary)

Emitted when the corresponding event type is enabled via `set events`.

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Always `"alloc_by_class"` |
| `tsNs` | long | Timestamp in nanoseconds |
| `pid` | int | Process ID |
| `sessionId` | string | Session identifier |
| `tid` | int | OS thread ID |
| `className` | string | Fully qualified type name |
| `count` | int | Number of allocations of this type |

```json
{"event":"alloc_by_class","tsNs":123556789,"pid":1234,"sessionId":"a1b2c3","tid":5678,"className":"System.String","count":1234}
```

When ELT hooks are active, allocation events may include call-site attribution:

| Field | Type | Description |
|-------|------|-------------|
| `allocAsm` | string | *(optional)* Assembly of the allocating method |
| `allocNs` | string | *(optional)* Namespace of the allocating method |
| `allocCls` | string | *(optional)* Class of the allocating method |
| `allocM` | string | *(optional)* Method name of the allocating method |

```json
{"event":"alloc_by_class","tsNs":123556789,"pid":1234,"sessionId":"a1b2c3","tid":5678,"className":"System.String","count":42,"allocAsm":"MyApp","allocNs":"MyApp.Services","allocCls":"UserService","allocM":"GetUser"}
```

### `contention_start` (lock contention started)

Emitted via EventPipe when `contention_start` event type is enabled. Requires .NET 5+ runtime.

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Always `"contention_start"` |
| `tsNs` | long | Timestamp in nanoseconds |
| `pid` | int | Process ID |
| `sessionId` | string | Session identifier |
| `tid` | int | OS thread ID |

```json
{"event":"contention_start","tsNs":123456789,"pid":1234,"sessionId":"a1b2c3","tid":5678}
```

### `contention_end` (lock contention ended)

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Always `"contention_end"` |
| `tsNs` | long | Timestamp in nanoseconds |
| `pid` | int | Process ID |
| `sessionId` | string | Session identifier |
| `tid` | int | OS thread ID |
| `durationNs` | long | Duration the thread was blocked waiting for the lock |

```json
{"event":"contention_end","tsNs":123556789,"pid":1234,"sessionId":"a1b2c3","tid":5678,"durationNs":5000000}
```

## Field Semantics

| Field | Meaning |
|-------|---------|
| `tsNs` | Monotonic nanosecond timestamp from `QueryPerformanceCounter`. Monotonically increasing within a run. For `method_stats`/`exception_stats`, always `0` (emitted at shutdown). |
| `deltaNs` | Wall-clock nanoseconds from method enter to leave. Always non-negative. Includes time spent in child calls. |
| `depth` | Call stack depth. 0 = top-level entry point. Increases by 1 for each nested call. |
| `tid` | OS thread ID. Use to separate interleaved events from concurrent threads. |
| `async` | `true` when the method is an async state machine `MoveNext`. The `m` field contains the original method name (not `MoveNext`). |
| `sessionId` | Session identifier. Matches the `session_metadata` event. |
| `totalSelfNs` | Sum of self-time (excluding children) across all calls. Key metric for identifying CPU-bound bottlenecks. |
| `totalInclusiveNs` | Sum of inclusive-time (including children) across all calls. High inclusive with low self = orchestrator method. |
| `callCount` | Total invocations. High count with modest per-call time = "death by a thousand cuts" pattern. |

## Async Method Interpretation

Async methods in .NET compile to state machine classes named `<MethodName>d__N`. The profiler resolves these automatically:

- The `m` field shows the **original method name** (e.g., `ProcessOrderAsync`), not `MoveNext`
- The `cls` field may show the state machine class name (e.g., `<ProcessOrderAsync>d__5`)
- `async` is `true` for these events
- **Continuations may appear on different thread IDs** than the initial call — filter by method name, not just tid, when tracing async flows
- A single async method call may produce multiple enter/leave pairs (one per state machine step)

## Analysis Algorithms

### Discovery Analysis (from `method_stats`)

1. Extract all `method_stats` events: `grep '"event":"method_stats"' discovery-*.ndjson`
2. Parse JSON, sort by `totalSelfNs` descending — identifies where CPU time is actually spent
3. Compare `totalSelfNs` vs `totalInclusiveNs`:
   - **Self ~ Inclusive** — leaf method, the work is here
   - **Self << Inclusive** — orchestrator, drill into children with targeted tracing
4. Check `callCount` — high count with modest per-call time suggests batching/caching opportunity
5. Check `maxSelfNs` vs `totalSelfNs / callCount` — large discrepancy indicates occasional spikes

### Exception Discovery (from `exception_stats`)

1. Extract all `exception_stats` events: `grep '"event":"exception_stats"' discovery-*.ndjson`
2. Sort by `count` descending — most frequently thrown exceptions
3. Group by `exType` to find systemic exception patterns
4. High-count exceptions in hot paths indicate control flow via exceptions (anti-pattern)

### Performance Hotspot Detection (from `enter`/`leave`)

1. Extract all `leave` events: `grep '"event":"leave"' trace.ndjson`
2. Parse JSON, sort by `deltaNs` descending
3. Take top-20 entries — these are the slowest individual method calls
4. Group by `asm.ns.cls.m` to find methods that are consistently slow vs. one-off spikes

### Exception Tracing (from `exception` events)

1. Find all `exception` events: `grep '"event":"exception"' trace.ndjson`
2. For each exception, note the line number in the file
3. Read 50 preceding lines to see the call stack leading to the exception
4. Use `tid` to filter — only events on the same thread are part of that call stack

### Call Tree Reconstruction

1. Filter events by a single `tid`
2. Process in order: `enter` increases depth, `leave` decreases depth
3. Use `depth` to build indentation: `depth * 2 spaces` per level
4. `exception` events mark where the normal flow broke

### Method Aggregation (for `analyze-diff`)

1. Create a key from `asm + "." + ns + "." + cls + "." + m`
2. Sum all `deltaNs` values for that key across the trace
3. Compare totals between base and comparison traces

## Timing Thresholds

| deltaNs Range | Human-Readable | Assessment |
|---------------|---------------|------------|
| < 1,000 ns | < 1 us | Trivial — ignore |
| 1,000 - 1,000,000 ns | 1 us - 1 ms | Normal method execution |
| 1,000,000 - 100,000,000 ns | 1 ms - 100 ms | Worth investigating |
| > 100,000,000 ns | > 100 ms | Critical hotspot — prioritize |

### Context for Thresholds

- **API endpoint latency budget**: typically 50-200 ms total
- **UI frame budget**: 16.7 ms (60 fps)
- **Database query**: 1-50 ms is typical; > 100 ms is slow
- **Serialization/deserialization**: usually < 5 ms unless large payloads

## Validation Rules

The NDJSON output follows these invariants (enforced by `test/validate.py`):

1. First event must be `session_metadata`
2. Each line must be valid JSON
3. All required fields must be present for each event type
4. Timestamps must be monotonically non-decreasing (except `method_stats`/`exception_stats` which have `tsNs: 0`)
5. `deltaNs` values must be non-negative
6. Enter/leave events should balance per thread (accounting for exceptions breaking the flow)
