# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Unreleased

## [0.1.4] - 2026-03-05

### Changed

- **Events system replaces trackMemory** — the `events` config field is now the sole mechanism for controlling event types. `set track-memory` and `set mode` commands have been removed. Use `set events` with specific event types instead (e.g., `gc_start gc_end alloc_by_class` for memory analysis).
- **Two-phase profiling workflow** — SKILL.md now recommends starting with lightweight `method_stats` discovery sessions before targeted `enter`/`leave` tracing. This reduces output volume while identifying hotspot areas.
- **Phase 4 restructured** — Analysis phase now covers: interpreting discovery results (grep/python on `method_stats`), deciding next action, targeted analysis with CLI commands, and iterative drill-down with an Events column in the strategy table.

### Added

- `method_stats` event schema in ndjson-reference.md — aggregated per-method timing (callCount, totalSelfNs, maxSelfNs, totalInclusiveNs, maxInclusiveNs)
- `exception_stats` event schema in ndjson-reference.md — aggregated per-exception-type counts
- Event Categories table classifying all events by category, emission timing, and maxEvents behavior
- Discovery analysis algorithms for interpreting `method_stats` and `exception_stats` data
- `set events` command in CLI Quick Reference

### Removed

- `set track-memory` from CLI Quick Reference and session setup
- `set mode` from CLI Quick Reference
- All `trackMemory` references from skill and NDJSON reference

### Added (initial)

- Initial plugin release with metreja-profiler skill
- NDJSON schema reference documentation
- CI/CD pipeline with GitVersion and marketplace sync
