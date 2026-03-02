---
name: metreja-profiler
description: Use when profiling .NET applications for performance bottlenecks, slow methods, execution tracing, or debugging exceptions through call-path analysis instead of manual logging
---

# Metreja .NET Call-Path Profiler

## Tool Installation

| Asset | Details |
|-------|---------|
| CLI | `metreja` (installed as .NET global tool) |
| Package | `Metreja` on NuGet |
| CLSID | `{7C8F944B-4810-4999-BF98-6A3361185FC2}` |

The native profiler DLL is bundled inside the tool package and auto-discovered at runtime — no manual path configuration needed.

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
dotnet tool install -g Metreja
```

## Phase 1: Target Analysis & Filter Design

1. **Identify the target project** — find the `.csproj` file and extract the assembly name (usually `<AssemblyName>` or the project file name without extension)
2. **Choose a filter strategy** based on the use case:

| Use Case | Include Filters | Exclude Filters | max-events | track-memory |
|----------|----------------|-----------------|------------|--------------|
| Broad performance scan | `--assembly "AppName"` | (defaults: `System.*`, `Microsoft.*`) | 50000 | false |
| Focused class perf | `--class "ClassName"` | (defaults) | 50000 | false |
| Debug specific flow | `--namespace "Ns"` | clear defaults if needed | 100000 | false |
| Debug exception | `--assembly "AppName"` | clear defaults if needed | 100000 | false |
| Memory / GC analysis | `--assembly "AppName"` | (defaults) | 50000 | **true** |

**Filter rules** target a single level — each `add include`/`add exclude` call takes exactly one of `--assembly`, `--namespace`, `--class`, or `--method` (mutually exclusive). Multiple patterns can be passed to the same option (e.g., `--assembly "A" "B"`).

**Default excludes:** New sessions automatically exclude `System.*` and `Microsoft.*` assemblies. Use `metreja clear-filters -s ID --excludes` to remove them if needed.

**Filter patterns** support `*` as wildcard (e.g., `System.*` matches `System.IO`, `System.Linq`, etc.).

## Phase 2: Session Setup

Run these commands sequentially. Each depends on the session ID from step 1.

```bash
# 1. Create session — capture the printed session ID
#    (automatically adds default excludes: System.*, Microsoft.*)
SESSION=$(metreja init --scenario "perf-investigation")

# 2. Add include filter (assembly name from Phase 1)
metreja add include -s $SESSION --assembly "MyApp"

# 3. (Optional) Add extra exclude filters — defaults already cover System.*/Microsoft.*
# metreja add exclude -s $SESSION --assembly "SomeOtherLib.*"

# 4. Set output path (use tokens for unique filenames)
metreja set output -s $SESSION "trace-{sessionId}-{pid}.ndjson"

# 5. Enable delta timing (required for performance analysis)
metreja set compute-deltas -s $SESSION true

# 6. Set max events cap
metreja set max-events -s $SESSION 50000

# 7. Enable memory tracking (for GC/allocation analysis)
metreja set track-memory -s $SESSION true

# 8. Validate the session configuration
metreja validate -s $SESSION
```

Validation checks: `sessionId` exists, `output.path` set, at least one include rule, output directory writable. Fix any reported errors before proceeding.

## Phase 3: Profiled Execution

### Strategy A: Short-lived apps (console apps, tests)

Build the project and run with profiling — the `run` command injects all profiler env vars automatically:

```bash
# Build the project
dotnet build <target-project-path> -c Debug

# Run with profiling (waits for exit)
metreja run -s $SESSION "<path-to-built-exe>"
```

### Strategy B: Long-running / GUI apps (WPF, web servers, services)

For apps that don't exit on their own, use `--detach` to launch and return immediately:

```bash
# Build the project
dotnet build <target-project-path> -c Debug

# Launch with profiling (returns immediately)
metreja run -s $SESSION --detach "<path-to-built-exe>"
# Tell user to exercise the app and close it when done
# Wait for user signal, then proceed to Phase 4
```

### Required Environment Variables (set automatically by `metreja run`)

| Variable | Value |
|----------|-------|
| `CORECLR_ENABLE_PROFILING` | `1` |
| `CORECLR_PROFILER` | `{7C8F944B-4810-4999-BF98-6A3361185FC2}` |
| `CORECLR_PROFILER_PATH` | Absolute path to `Metreja.Profiler.dll` (auto-resolved) |
| `METREJA_CONFIG` | Absolute path to session JSON (`.metreja/sessions/<id>.json`) |

## Phase 4: Analysis

Use the built-in analysis commands — they handle streaming, self-time calculation, and formatting. Reference [ndjson-reference.md](ndjson-reference.md) for event schemas.

### Step 1: Hotspots overview

```bash
# Top-10 hotspots with self time (shows where time is actually spent)
metreja hotspots trace.ndjson --top 10

# Sort by inclusive time (shows total wall-clock per method)
metreja hotspots trace.ndjson --top 10 --sort inclusive

# Sort by call count (most-called methods first)
metreja hotspots trace.ndjson --top 10 --sort calls

# Sort by allocations (requires track-memory enabled; shows Allocs column)
metreja hotspots trace.ndjson --top 10 --sort allocs

# Filter out noise below 1ms
metreja hotspots trace.ndjson --top 10 --min-ms 1
```

Present the hotspot table to the user. Self time distinguishes orchestrators (high inclusive, low self) from actual work (high self).

### Step 2: Drill into slow methods

For each suspicious method from the hotspots, drill deeper:

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

### Step 3: Scope-narrowed analysis

Instead of limiting output, run a tighter-scope analysis with `--filter`:

```bash
# Focus hotspots on a specific class or namespace
metreja hotspots trace.ndjson --filter "SlowClass"
metreja hotspots trace.ndjson --filter "MyApp.Data" --filter "MyApp.Services"
```

### Step 4: Memory analysis (when `track-memory` is enabled)

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

**When analysis reveals the bottleneck is in a specific class/namespace but you need more detail**, create a new profiling session with tighter filters and re-profile. This captures more events within the area of interest.

```bash
# Create a new drill-down session with narrower scope
# (default excludes System.*/Microsoft.* are added automatically)
DRILL_SESSION=$(metreja init --scenario "drill-down-1")
metreja add include -s $DRILL_SESSION --class "SlowClass"
metreja set output -s $DRILL_SESSION "trace-drill-{sessionId}-{pid}.ndjson"
metreja set compute-deltas -s $DRILL_SESSION true
metreja set max-events -s $DRILL_SESSION 50000
metreja validate -s $DRILL_SESSION

# Build and re-profile with the narrower session (same Phase 3 workflow)
dotnet build <target-project-path> -c Debug
metreja run -s $DRILL_SESSION "<path-to-built-exe>"

# Analyze the drill-down trace
metreja hotspots trace-drill-*.ndjson --top 20
metreja calltree trace-drill-*.ndjson --method "SuspiciousMethod"
```

**Drill-down strategy:**

| Iteration | Filter Level | When to use |
|-----------|-------------|-------------|
| 1 (broad) | `--assembly "AppName"` | Initial discovery |
| 2 | `--namespace "Slow.Namespace"` | Hotspot points to a namespace |
| 3 | `--class "SlowClass"` | Hotspot points to a specific class |
| 4 | `--method "SlowMethod"` | Need specific method-level detail |

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
   This outputs a table comparing total time per method between the two runs.

3. **Cleanup** — delete the session when done:
   ```bash
   metreja clear -s $SESSION
   ```

## CLI Quick Reference

| Command | Syntax | Purpose |
|---------|--------|---------|
| `init` | `metreja init [--scenario NAME]` | Create session, prints session ID |
| `add include` | `metreja add include -s ID --assembly\|--namespace\|--class\|--method P [P2...]` | Add include filter (one level per call) |
| `add exclude` | `metreja add exclude -s ID --assembly\|--namespace\|--class\|--method P [P2...]` | Add exclude filter (one level per call) |
| `set output` | `metreja set output -s ID PATH` | Set output path (supports `{sessionId}`, `{pid}` tokens) |
| `set compute-deltas` | `metreja set compute-deltas -s ID true\|false` | Enable/disable delta timing |
| `set max-events` | `metreja set max-events -s ID N` | Cap event count (0 = unlimited) |
| `set metadata` | `metreja set metadata -s ID [--scenario S]` | Update scenario |
| `set track-memory` | `metreja set track-memory -s ID true\|false` | Enable/disable GC and allocation tracking |
| `set mode` | `metreja set mode -s ID MODE` | Set instrumentation mode (`elt3`) |
| `validate` | `metreja validate -s ID` | Validate session config |
| `run` | `metreja run -s ID [--detach] EXE [ARGS...]` | Launch exe with profiler env vars attached |
| `generate-env` | `metreja generate-env -s ID [--dll-path P] [--format batch\|powershell]` | Generate env var script for manual use |
| `analyze-diff` | `metreja analyze-diff BASE COMPARE` | Compare two NDJSON traces |
| `hotspots` | `metreja hotspots FILE [--top N] [--min-ms N] [--sort self\|inclusive\|calls\|allocs] [--filter PAT]...` | Per-method timing hotspots with self time and allocs |
| `calltree` | `metreja calltree FILE --method PAT [--tid N] [--occurrence N]` | Call tree for a specific method invocation |
| `callers` | `metreja callers FILE --method PAT [--top N]` | Who calls a specific method, with timing |
| `memory` | `metreja memory FILE [--top N] [--filter PAT]...` | GC summary and allocation hotspots by class |
| `clear` | `metreja clear -s ID \| --all` | Delete session(s) |

## Common Pitfalls

- **Shell state doesn't persist between Bash tool calls.** Use `metreja run` to launch profiled apps — it injects env vars via `ProcessStartInfo` without needing batch scripts. For manual use, `generate-env` can create a sourceable script.
- **`COR_PRF_ENABLE_FRAME_INFO`** is already set by the DLL in its event mask — no user action needed.
- **Large traces blow up context.** Always set `max-events` (50k for perf, 100k for debugging). Never read an entire large NDJSON file — use grep/python to extract relevant events.
- **Async methods** appear as `<MethodName>d__N` state machine classes. The profiler resolves these: the `m` field shows the original method name, and `async` is `true`. Continuations may appear on different thread IDs than the initial call.
- **Output path must be writable.** The directory in `set output` path must exist or the profiler will fail silently. Create it before running.
- **Session config location.** Config JSON lives at `.metreja/sessions/<session-id>.json` relative to the working directory where you ran `metreja init`.
