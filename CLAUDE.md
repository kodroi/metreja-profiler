# Metreja Profiler Plugin

Claude Code plugin for .NET Call-Path Profiling.

## Project Structure

- `skills/metreja-profiler/SKILL.md` — Main skill guide (phases, CLI reference, analysis workflow)
- `skills/metreja-profiler/ndjson-reference.md` — NDJSON event schema reference
- `.claude-plugin/plugin.json` — Plugin metadata and version

## Testing

This is a skill-only plugin — no executable code to test. Validate with:

1. `plugin.json` is valid JSON: `python -c "import json; json.load(open('.claude-plugin/plugin.json'))"`
2. `SKILL.md` exists and has the frontmatter header
3. `ndjson-reference.md` exists

## Versioning

- Uses [GitVersion](https://gitversion.net/) with `ContinuousDeployment` mode
- Bump with commit message suffixes: `+semver: minor`, `+semver: patch`, `+semver: major`
- CI updates `plugin.json` version and syncs to the marketplace repo on tag

## Changelog

When making changes, add an entry under `## Unreleased` in `CHANGELOG.md` following [Keep a Changelog](https://keepachangelog.com/) format.
