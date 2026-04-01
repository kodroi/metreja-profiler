# Metreja Profiler

**A .NET profiler plugin for Claude Code — built for AI-driven performance and memory analysis.** Tell Claude "find why this is slow" or "where am I wasting memory?" and it profiles, analyzes, and gives you an answer backed by real data. No detours, no GUIs.

> "This endpoint takes 3 seconds, find out why" / "Check if the import service is leaking memory" — Claude handles session setup, profiling, and analysis automatically. Both slowness and wastefulness, one conversation.

## What You Get

Install the plugin and Claude profiles as part of fixing the problem — not as a separate step. The skill activates automatically, runs the full measure-analyze-fix loop, and gives you actionable results:

```
You: "This endpoint takes 3 seconds, find out why"

Claude: Setting up profiling session...
        Running with profiler attached...
        Analyzing trace (42,381 events captured)...

        Top hotspots (by self time):
        ┌──────────────────────────────────┬───────┬────────────┬───────────┐
        │ Method                           │ Calls │ Self (ms)  │ Incl (ms) │
        ├──────────────────────────────────┼───────┼────────────┼───────────┤
        │ OrderService.ValidateInventory   │    47 │    1,842.3 │   2,104.1 │
        │ DbContext.SaveChangesInternal    │    12 │      891.7 │     891.7 │
        │ PricingEngine.CalculateDiscount  │   312 │      203.4 │     247.8 │
        └──────────────────────────────────┴───────┴────────────┴───────────┘

        ValidateInventory is your bottleneck — 1.8s of self time across
        47 calls. It queries inventory per-item instead of batching.
        Here's the fix...
```

## Plugin Contents

| Component | Name | Description |
|-----------|------|-------------|
| Skill | `metreja-profiler` | Structured .NET profiling workflow — auto-activates on performance, memory, and debugging questions |

## Installation

### Prerequisites

[Metreja CLI](https://www.nuget.org/packages/Metreja.Tool) — the .NET global tool that does the actual profiling:

```bash
dotnet tool install -g Metreja
```

### Add Marketplace

```
/plugin marketplace add kodroi/metreja-profiler-marketplace
```

### Install Plugin

```
/plugin install metreja-profiler@metreja-profiler-marketplace
```

## Capabilities

**Finds the real bottleneck** — Self-time analysis pinpoints the method that's actually slow, not the one that calls it.

**Catches wastefulness** — Excessive allocations, GC thrashing, memory pressure. See which types allocate the most and which methods trigger gen2 collections.

**Traces call paths** — Follow execution through your code, including async state machines resolved to their original method names.

**Proves the fix worked** — Diff two traces. See the numbers. No guessing whether your change helped.

**Drills down iteratively** — Start broad, then narrow to a specific namespace, class, or method. The agent handles the iterative investigation.

## How It Works

The skill guides Claude through five phases:

1. **Target analysis** — Identifies the project and designs method filters based on your question
2. **Session setup** — Creates a profiling session with the `metreja` CLI, configuring filters, output paths, and event caps
3. **Profiled execution** — Runs your app with the native profiler DLL attached via .NET's `COR_PROFILER` mechanism
4. **Analysis** — Reads the captured NDJSON trace using hotspots, call tree, callers, and memory commands
5. **Findings** — Links trace data back to your source code with actionable fix suggestions

## Example Prompts

| What you say | What happens |
|---|---|
| "Profile this app and find the bottleneck" | Full performance scan with hotspot analysis |
| "Why is `ProcessOrder` slow?" | Targeted profiling with call tree drill-down |
| "Check for memory leaks in the import service" | GC + allocation analysis with `track-memory` enabled |
| "What exceptions are thrown during checkout?" | Execution trace with exception event filtering |
| "Compare performance before and after my fix" | Two profiling sessions + `analyze-diff` |

## CLI Quick Reference

| Command | Purpose |
|---------|---------|
| `metreja init` | Create a profiling session |
| `metreja add include/exclude` | Add method filters (assembly, namespace, class, method) |
| `metreja set output` | Set trace output path (supports `{sessionId}`, `{pid}` tokens) |
| `metreja set compute-deltas` | Enable delta timing for performance analysis |
| `metreja set disable-inlining` | Control JIT inlining (default: enabled for realistic profiling) |
| `metreja set disable-optimizations` | Control JIT optimizations (default: enabled) |
| `metreja set track-memory` | Enable GC and allocation tracking |
| `metreja validate` | Validate session config before profiling |
| `metreja generate-env` | Generate environment variable script (batch or PowerShell) |
| `metreja hotspots` | Find hotspots by self time, inclusive time, calls, or allocations |
| `metreja calltree` | View call tree for a specific method invocation |
| `metreja callers` | See who calls a method, with timing breakdown |
| `metreja memory` | GC summary and allocation hotspots by type |
| `metreja analyze-diff` | Compare two traces side by side |
| `metreja clear` | Delete session(s) |

## Troubleshooting

### "metreja: command not found"

Install the CLI as a .NET global tool:

```bash
dotnet tool install -g Metreja
```

Ensure `~/.dotnet/tools` is on your PATH.

### Profiler produces no output

- Check that the output directory exists and is writable
- Verify filters aren't too restrictive — start with `--assembly "YourApp"` only
- Confirm the app is running on .NET (Core) — the profiler doesn't support .NET Framework

### Trace is too large / too many events

Set a cap during session setup: `metreja set max-events -s $SESSION 50000`. For focused analysis, use tighter include filters (namespace or class level).

### Async methods show as state machines

This is expected — the profiler resolves `<MethodName>d__N` back to the original method name. The `async: true` flag in the trace identifies these.

## Requirements

- Windows 10/11 (x64) — the native profiler DLL is Windows-only
- .NET 8, 9, or 10 runtime
- [Metreja CLI](https://www.nuget.org/packages/Metreja.Tool) global tool
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI

## License

MIT
