# Changelog

## [1.0.1] — 2026-03-13

Bugfix release. No feature changes.

### Fixed
- **Self-protection**: killall no longer targets or kills its own running process. Previously, a regex or wildcard pattern could match the killall.exe process itself, causing Windows to truncate the EXE to 0 bytes.

### Added
- `--version` / `-V` flag to display the current version.
- Version number (`1.0.1`) embedded in the assembly metadata.


---

## [1.0.0] — 2026-03-12

Initial release.

- Pattern-based process termination (exact, substring, glob, regex).
- Advanced filters: `--cmdline`, `--module`, `--port`, `--window`, `--parent`.
- Process tree kill (`--tree`).
- Three-tier safety model (Immortal, Auto-Restart, Allowed).
- Subcommands: `hung`, `networkapps`, `ramhog`, `cpuhog`, `gpu`, `llm`, `game`, `restart`.
- Dry-run and force modes.
- Self-contained, single-file portable build.
