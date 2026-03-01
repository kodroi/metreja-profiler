# Metreja Profiler

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin for profiling .NET applications — find performance bottlenecks, trace execution flow, and debug exceptions through call-path analysis.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [Metreja](https://www.nuget.org/packages/Metreja) .NET global tool:
  ```bash
  dotnet tool install -g Metreja
  ```

## Installation

### Via Marketplace (recommended)

```
/plugin marketplace add kodroi/metreja-profiler-marketplace
/plugin install metreja-profiler@metreja-profiler-marketplace
```

### Direct Install

```
/plugin install kodroi/metreja-profiler
```

## Usage

Once installed, the skill activates automatically when you ask Claude Code about:

- **Performance**: "Profile this app", "Find the bottleneck", "Why is this slow?"
- **Memory**: "Check for memory leaks", "Analyze GC pressure"
- **Debugging**: "Trace the execution flow", "What exceptions are thrown?"

The skill guides Claude through a structured workflow:

1. **Target Analysis** — Identify the project and design filters
2. **Session Setup** — Configure profiling session with `metreja` CLI
3. **Profiled Execution** — Run the app with profiling enabled
4. **Analysis** — Hotspots, call trees, callers, memory analysis
5. **Findings** — Actionable insights linked back to source code

## CLI Quick Reference

| Command | Purpose |
|---------|---------|
| `metreja init` | Create a profiling session |
| `metreja add include/exclude` | Add method filters |
| `metreja set output` | Set trace output path |
| `metreja validate` | Validate session config |
| `metreja generate-env` | Generate environment variable script |
| `metreja hotspots` | Find performance hotspots |
| `metreja calltree` | View call tree for a method |
| `metreja callers` | See who calls a method |
| `metreja memory` | GC summary and allocation hotspots |
| `metreja analyze-diff` | Compare two traces |

## License

MIT
