---
name: metreja-profiler
description: Use when profiling .NET applications for performance bottlenecks, slow methods, execution tracing, or debugging exceptions through call-path analysis instead of manual logging
---

# Metreja .NET Call-Path Profiler

## Tool Installation

| Asset | Details |
|-------|---------|
| CLI | `metreja` (installed as .NET global tool) |
| Package | `Metreja.Tool` on NuGet |
| CLSID | `{7C8F944B-4810-4999-BF98-6A3361185FC2}` |

The native profiler library (`Metreja.Profiler.dll` on Windows, `libMetreja.Profiler.dylib` on macOS) is bundled inside the tool package and auto-discovered at runtime — no manual path configuration needed.

## Use-Case Detection

Activate this skill when the user's request matches any category:

```
User request
  |
  |-- Contains performance keywords? --> Performance profiling
  |     (slow, optimize, bottleneck, performance, latency, profile)
  |
  |-- Contains memory keywords? --> Memory profiling
  |     (memory, GC, garbage collection, allocation, heap, leak)
  |
  |-- Contains debugging keywords? --> Execution tracing
        (trace, exception, bug, flow, call path, what's happening, logging)
```

## Phase 0: Prerequisites

Verify the tool is installed:

```bash
dotnet tool list -g | grep -i metreja
```

If missing, install it:

```bash
dotnet tool install -g Metreja.Tool
```

## Phase 1: Target Analysis & Filter Design

1. **Identify the target project** — find the `.csproj` file and extract the assembly name (usually `<AssemblyName>` or the project file name without extension)

2. **Understand event types** — Metreja uses an `events` config field to control what data the profiler emits:

| Category | Event Types | Description | maxEvents behavior |
|----------|-------------|-------------|--------------------|
| Aggregated | `method_stats`, `exception_stats` | In-profiler summaries emitted periodically and at shutdown | Bypass maxEvents cap |
| Per-call | `enter`, `leave`, `exception` | One event per method call/exit/throw | Count against maxEvents |
| Memory | `gc_start`, `gc_end`, `alloc_by_class` | GC and allocation tracking | `gc_*` bypass cap; `alloc_by_class` counts |

**Two-phase approach:** Start with `method_stats` for lightweight discovery (low output, same ELT3 hooks), then use `enter`/`leave` for targeted tracing on identified hotspots.

3. **Choose a filter and events strategy** based on the use case:

| Use Case | Include Filters | Exclude Filters | Events | max-events |
|----------|----------------|-----------------|--------|------------|
| Broad performance discovery | `--assembly "AppName"` | `System.*`, `Microsoft.*` | `method_stats exception_stats` | not needed (stats bypass cap) |
| Focused method perf | `--class "ClassName"` | `System.*`, `Microsoft.*` | `enter leave exception` | 50000 |
| Debug specific flow | `--namespace "Ns"` | (none) | `enter leave exception` | 100000 |
| Debug exception | `--assembly "AppName"` | (none) | `enter leave exception exception_stats` | 100000 |
| Memory / GC analysis | `--assembly "AppName"` | `System.*`, `Microsoft.*` | `gc_start gc_end alloc_by_class` | 50000 |

**Filter patterns** support `*` as wildcard (e.g., `System.*` matches `System.IO`, `System.Linq`, etc.).

4. **Understand filter evaluation logic:**
   - A method is traced if it matches **any** include rule AND does **not** match **any** exclude rule
   - Each `add include`/`add exclude` command accepts exactly **one level** (`--assembly`, `--namespace`, `--class`, or `--method`) but can take **multiple patterns** at that level (e.g., `--namespace "A" --namespace "B"`)
   - Filters are evaluated at JIT time via `FunctionIDMapper2` — excluded methods get zero ELT3 overhead (no per-call cost)
   - **Example — exclude high-frequency API clients within an included assembly:**
     ```bash
     metreja add include -s $SESSION --assembly "MyApp"
     metreja add exclude -s $SESSION --namespace "MyApp.Generated"
     metreja add exclude -s $SESSION --class "*ApiClient"
     ```

## Phase 2: Session Setup

Run these commands sequentially. Each depends on the session ID from step 1.

### Discovery session (recommended first step)

```bash
# 1. Create session — capture the printed session ID
SESSION=$(metreja init --scenario "discovery")

# 2. Add include filter (assembly name from Phase 1)
metreja add include -s $SESSION --assembly "MyApp"

# 3. Add exclude filters
metreja add exclude -s $SESSION --assembly "System.*"
metreja add exclude -s $SESSION --assembly "Microsoft.*"

# 4. Set output path (use tokens for unique filenames)
metreja set output -s $SESSION "discovery-{sessionId}-{pid}.ndjson"

# 5. Enable delta timing (required for performance analysis)
metreja set compute-deltas -s $SESSION true

# 6. Set events — aggregated stats for lightweight discovery
metreja set events -s $SESSION method_stats exception_stats

# 7. Validate the session configuration
metreja validate -s $SESSION
```

### Targeted tracing session (after discovery identifies hotspots)

```bash
# 1. Create session with narrowed scope
TRACE_SESSION=$(metreja init --scenario "targeted-trace")

# 2. Add narrowed include filter (class/method from discovery)
metreja add include -s $TRACE_SESSION --class "SlowClass"

# 3. Add exclude filters
metreja add exclude -s $TRACE_SESSION --assembly "System.*"
metreja add exclude -s $TRACE_SESSION --assembly "Microsoft.*"

# 4. Set output path
metreja set output -s $TRACE_SESSION "trace-{sessionId}-{pid}.ndjson"

# 5. Enable delta timing
metreja set compute-deltas -s $TRACE_SESSION true

# 6. Set events — per-call tracing for detailed analysis
metreja set events -s $TRACE_SESSION enter leave exception

# 7. Set max events cap (per-call events count against this)
metreja set max-events -s $TRACE_SESSION 50000

# 8. Validate
metreja validate -s $TRACE_SESSION
```

Validation checks: `sessionId` exists, `output.path` set, at least one include rule, output directory writable. Fix any reported errors before proceeding.

## Phase 3: Profiled Execution

### Strategy A: Short-lived apps (console apps, tests)

Generate the env script then source it and run inline. Use `$SESSION` for discovery or `$TRACE_SESSION` for targeted tracing:

```bash
# Generate env vars as shell script (DLL path is auto-discovered)
# Use $SESSION for discovery, or $TRACE_SESSION for targeted tracing
metreja generate-env -s $SESSION --format shell > env.sh

# Source and run:
source env.sh && dotnet run --project <target-project-path> -c Release
```

The session config JSON is at `.metreja/sessions/<SESSION>.json` relative to the working directory.

### Strategy B: Long-running apps (web servers, services)

For apps that don't exit on their own:

1. Generate the env script: `metreja generate-env -s $SESSION --format powershell`
2. Print the env vars to the user with instructions to paste into their terminal
3. Tell the user to run their app manually
4. While the app is running, you can trigger a manual stats flush to inspect results without stopping:
   ```bash
   metreja flush --pid PID
   ```
   This signals the profiler to write current delta `method_stats`/`exception_stats` to the NDJSON file immediately. Obtain the PID from the output filename (if `{pid}` token was used) or via `ps`/Task Manager.
5. Wait for the user to signal that the app has been exercised and stopped
6. Proceed to Phase 4 with the generated NDJSON file

### Required Environment Variables

| Variable | Value |
|----------|-------|
| `CORECLR_ENABLE_PROFILING` | `1` |
| `CORECLR_PROFILER` | `{7C8F944B-4810-4999-BF98-6A3361185FC2}` |
| `CORECLR_PROFILER_PATH` | Absolute path to `Metreja.Profiler.dll` (Windows) or `libMetreja.Profiler.dylib` (macOS) — auto-resolved by `generate-env` |
| `METREJA_CONFIG` | Absolute path to session JSON (`.metreja/sessions/<id>.json`) |

## Phase 4: Analysis

Use the built-in analysis commands and grep/python for discovery data. Reference [ndjson-reference.md](ndjson-reference.md) for event schemas.

### Step 1: Interpret discovery results (from `method_stats`/`exception_stats` sessions)

Discovery sessions emit aggregated stats. Use `metreja hotspots` directly — it reads both `method_stats` and `enter`/`leave` events:

```bash
# Top methods by total self-time (where CPU time is actually spent)
metreja hotspots discovery-*.ndjson --top 20

# Sort by inclusive time, call count, etc.
metreja hotspots discovery-*.ndjson --top 20 --sort inclusive
metreja hotspots discovery-*.ndjson --top 20 --sort calls

# Filter to specific namespace
metreja hotspots discovery-*.ndjson --top 20 --filter "MyApp.Services"
```

For exception stats (not covered by `hotspots`), use grep/python:

```bash
# Top exception hotspots
grep '"event":"exception_stats"' discovery-*.ndjson | python3 -c "
import sys, json
exceptions = [json.loads(l) for l in sys.stdin]
exceptions.sort(key=lambda e: e['count'], reverse=True)
for e in exceptions[:10]:
    print(f\"  {e['count']:6d}x  {e['exType']}  in  {e['ns']}.{e['cls']}.{e['m']}\")
"
```

Present the discovery results to the user. Key interpretation:
- **High self-time, low inclusive-time** — the method itself is slow (CPU-bound work, I/O wait)
- **High inclusive-time, low self-time** — orchestrator calling slow children (drill into children)
- **High call count with modest per-call time** — death by a thousand cuts (consider caching/batching)

### Step 2: Decide next action

Based on discovery results:
- **Leaf method is the bottleneck** (high self-time) — locate in source code and suggest optimizations directly
- **Orchestrator method** (high inclusive, low self) — create a targeted tracing session (loop back to Phase 2) with `enter leave exception` events and narrowed class/method filters to see the call tree
- **Exception hotspot** — create targeted session with `enter leave exception` to capture the call stack leading to the exception

### Step 3: Targeted analysis (from `enter`/`leave` sessions)

These commands work on per-call trace data:

```bash
# Top-10 hotspots with self-time (shows where time is actually spent)
metreja hotspots trace.ndjson --top 10

# Sort by inclusive-time (shows total wall-clock per method)
metreja hotspots trace.ndjson --top 10 --sort inclusive

# Sort by call count (most-called methods first)
metreja hotspots trace.ndjson --top 10 --sort calls

# Sort by allocations (requires alloc_by_class events enabled)
metreja hotspots trace.ndjson --top 10 --sort allocs

# Filter out noise below 1ms
metreja hotspots trace.ndjson --top 10 --min-ms 1
```

Drill into slow methods:

```bash
# Call tree: shows what a method does internally (slowest invocation by default)
metreja calltree trace.ndjson --method "SlowMethod"

# Show 2nd slowest invocation
metreja calltree trace.ndjson --method "SlowMethod" --occurrence 2

# Filter by thread ID
metreja calltree trace.ndjson --method "SlowMethod" --tid 54576

# Who calls this method? Shows caller breakdown with timing
metreja callers trace.ndjson --method "SlowMethod"
```

Scope-narrowed analysis with `--filter`:

```bash
# Focus hotspots on a specific class or namespace
metreja hotspots trace.ndjson --filter "SlowClass"
metreja hotspots trace.ndjson --filter "MyApp.Data" --filter "MyApp.Services"
```

### Step 4: Memory analysis (when gc/alloc events are enabled)

```bash
# GC summary + top-20 allocation-by-class hotspots
metreja memory trace.ndjson

# Show top 50 allocation types
metreja memory trace.ndjson --top 50

# Filter to specific types
metreja memory trace.ndjson --filter "System.String" --filter "MyApp.Models"
```

The memory command shows:
- **GC Summary**: total GC count by generation (gen0/1/2), total/avg/max pause time
- **Allocation Hotspots**: per-class allocation counts sorted by count descending

Use this to identify: frequent gen2 collections (memory pressure), high allocation rates for specific types, and correlate GC pauses with call-path timing.

### Step 5: Iterative drill-down with new profiling sessions

**When analysis reveals the bottleneck is in a specific class/namespace but you need more detail**, create a new profiling session with tighter filters and re-profile.

**Drill-down strategy:**

| Iteration | Filter Level | Events | When to use |
|-----------|-------------|--------|-------------|
| 1 (discovery) | `--assembly "AppName"` | `method_stats exception_stats` | Initial discovery — identify hotspot areas |
| 2 (targeted) | `--namespace "Slow.Namespace"` | `enter leave exception` | Hotspot points to a namespace |
| 3 (focused) | `--class "SlowClass"` | `enter leave exception` | Hotspot points to a specific class |
| 4 (precise) | `--method "SlowMethod"` | `enter leave exception` | Need internal method-level detail |

**Keep drilling until:** the leaf method is identified (no further children), or the bottleneck is in external code (framework/DB/IO) where profiling won't help.

#### Exception tracing (when relevant)

```bash
grep '"event":"exception"' trace.ndjson | python3 -c "
import sys, json
for l in sys.stdin:
    e = json.loads(l)
    print(f\"  {e['exType']}  in  {e['ns']}.{e['cls']}.{e['m']}\")
"
```

## Phase 5: Findings & Cleanup

1. **Present actionable insights** — link trace data back to source code:
   - For each hotspot method, locate it in the codebase and suggest optimizations
   - For exceptions, show the call stack and the source of the exception

2. **Before/after comparison** (optional, if user optimizes and re-profiles):
   ```bash
   metreja analyze-diff base-trace.ndjson optimized-trace.ndjson
   ```
   This outputs a table comparing total self-time per method between the two runs, sorted by largest delta. Works with both `leave` events (per-call traces) and `method_stats` events (discovery sessions).

3. **Cleanup** — delete sessions when done:
   ```bash
   metreja clear -s $SESSION
   metreja clear -s $TRACE_SESSION   # if a targeted session was created
   ```

## CLI Quick Reference

| Command | Syntax | Purpose |
|---------|--------|---------|
| `init` | `metreja init [--scenario NAME]` | Create session, prints session ID |
| `add include` | `metreja add include -s ID [--assembly P] [--namespace P] [--class P] [--method P]` | Add include filter |
| `add exclude` | `metreja add exclude -s ID [--assembly P] [--namespace P] [--class P] [--method P]` | Add exclude filter |
| `set output` | `metreja set output -s ID PATH` | Set output path (supports `{sessionId}`, `{pid}` tokens) |
| `set compute-deltas` | `metreja set compute-deltas -s ID true\|false` | Enable/disable delta timing |
| `set max-events` | `metreja set max-events -s ID N` | Cap event count (0 = unlimited) |
| `set metadata` | `metreja set metadata -s ID [--scenario S]` | Update scenario |
| `set events` | `metreja set events -s ID TYPE [TYPE2...]` | Set enabled event types (`enter`, `leave`, `exception`, `method_stats`, `exception_stats`, `gc_start`, `gc_end`, `alloc_by_class`) |
| `set stats-flush-interval` | `metreja set stats-flush-interval -s ID SECONDS` | Periodic stats flush interval (default 30s, 0 = disabled). Protects against data loss on force-kill. |
| `validate` | `metreja validate -s ID` | Validate session config |
| `generate-env` | `metreja generate-env -s ID [--dll-path P] [--format batch\|powershell\|shell]` | Generate env var script (DLL path auto-detected) |
| `analyze-diff` | `metreja analyze-diff BASE COMPARE` | Compare two NDJSON traces |
| `hotspots` | `metreja hotspots FILE [--top N] [--min-ms N] [--sort self\|inclusive\|calls\|allocs] [--filter PAT]...` | Per-method timing hotspots with self-time and allocs |
| `calltree` | `metreja calltree FILE --method PAT [--tid N] [--occurrence N]` | Call tree for a specific method invocation |
| `callers` | `metreja callers FILE --method PAT [--top N]` | Who calls a specific method, with timing |
| `memory` | `metreja memory FILE [--top N] [--filter PAT]...` | GC summary and allocation hotspots by class |
| `clear` | `metreja clear -s ID \| --all` | Delete session(s) |
| `flush` | `metreja flush --pid PID` | Trigger manual stats flush on a running profiled process |
| `report` | `metreja report -t TITLE -d DESCRIPTION` | Report an issue to GitHub (requires `gh` CLI) |

## Reporting Issues

If you encounter a bug or unexpected behavior, report it directly from the CLI:

```bash
metreja report --title "Bug title" --description "Detailed description of what happened."
```

This creates a GitHub issue on the [kodroi/Metreja](https://github.com/kodroi/Metreja) repository. Requires the [GitHub CLI (`gh`)](https://cli.github.com/) installed and authenticated (`gh auth login`).

**Exit codes:** `0` = success, `2` = `gh` not installed, `3` = `gh` not authenticated, `1` = GitHub API error.

## Common Pitfalls

- **Start with discovery, not per-call tracing.** Use `method_stats` events first to identify hotspot areas with minimal output. Only switch to `enter`/`leave` events for targeted tracing after you know where to look.
- **Stats events bypass maxEvents.** `method_stats` and `exception_stats` are not subject to the `maxEvents` cap — they are emitted both periodically (per `statsFlushIntervalSeconds`, default 30s) and at final profiler shutdown. `gc_start`/`gc_end` also bypass the cap. Only `enter`, `leave`, `exception`, and `alloc_by_class` count against it.
- **Periodic stats flush protects against force-kill data loss.** By default (`statsFlushIntervalSeconds: 30`), the profiler periodically writes delta `method_stats`/`exception_stats` events to disk. If the profiled process is force-killed, you retain stats up to the last flush. Set to `0` to disable. For long-running processes or processes that may be killed, the default is recommended. The C# consumers (`hotspots`, `analyze-diff`) automatically sum multiple delta stats events.
- **Manual flush requires PID and stats events.** `metreja flush --pid PID` only works when the profiled process has `method_stats` or `exception_stats` events enabled. The PID can be obtained from the output filename (when the `{pid}` token is used) or via the OS process listing. The flush uses a named Windows event (`MetrejaFlush_{pid}`) or a POSIX named semaphore (`/MetrejaFlush_{pid}`) on macOS for inter-process signaling.
- **One filter level per command.** Each `add include`/`add exclude` command accepts only one of `--assembly`, `--namespace`, `--class`, or `--method`. To filter at multiple levels, use separate commands. Multiple patterns per level are allowed (e.g., `--namespace "A" --namespace "B"`).
- **`method_stats` still hooks ELT3.** The overhead is in output size, not execution speed — the profiler hooks every enter/leave regardless and aggregates in-process. Discovery sessions produce far fewer NDJSON lines but the profiled app runs at roughly the same speed.
- **Two output files with `dotnet run`.** When profiling via `dotnet run`, the .NET host process and the actual app are separate processes — you'll get two NDJSON files (one per PID). The host process file is usually tiny; use the larger file for analysis.
- **Shell state doesn't persist between Bash tool calls.** Always set env vars inline or use `generate-env --format shell` to create `env.sh` and source it in the same command (see Strategy A).
- **`COR_PRF_ENABLE_FRAME_INFO`** is already set by the DLL in its event mask — no user action needed.
- **Large traces blow up context.** Always set `max-events` for per-call tracing sessions (50k for perf, 100k for debugging). Never read an entire large NDJSON file — use grep/python to extract relevant events.
- **Async methods** appear as `<MethodName>d__N` state machine classes. The profiler resolves these: the `m` field shows the original method name, and `async` is `true`. Continuations may appear on different thread IDs than the initial call.
- **Output path must be writable.** The directory in `set output` path must exist or the profiler will fail silently. Create it before running.
- **Session config location.** Config JSON lives at `.metreja/sessions/<session-id>.json` relative to the working directory where you ran `metreja init`.
