# Oxide - Debug and Logging

This document describes the current debug tooling and logging behavior.

## Debug logging layers

Oxide has two separate debug/logging mechanisms.

### 1. Console lifecycle logs

Handled by `src/debug.rs`.

Characteristics:

- translated through `src/debug/i18n/`
- active only in debug builds
- printed to stdout with timestamps

Typical messages:

- application starting
- terminal ready
- application launched
- application exited
- audio backend events
- translation parse fallback issues

### 2. In-app debug terminal

Handled primarily from `src/app.rs` and rendered in `src/ui/debug_terminal.rs`.

Characteristics:

- detached viewport
- persistent live log buffer during the session
- searchable log output
- export to `.txt` / `.log`
- test report generation

## Debug terminal features

Current terminal features include:

- live log display
- search field
- export logs button
- test report button
- localized title and labels
- theme-aware rendering
- shortcut handling while terminal is focused

## Test report

`emit_test_report()` produces a compact diagnostics block intended for ROM testing.

Typical report content:

- trigger source
- ROM label
- running / paused / ROM loaded flags
- current PC / opcode / I / SP / timers
- lit pixel count
- keypad state summary
- quirk preset and individual flags
- full register dump

## Session log files

Oxide keeps rotating on-disk log files under:

- `logs/app/`
- `logs/emulator/`

Behavior on startup:

- if `latest.logs` exists, it is archived into a timestamped `.zip`
- a fresh `latest.logs` file is opened for the new session

## Exported logs vs rotating logs

These are different:

- rotating logs are automatic and session-based
- exported logs are user-triggered from the debug terminal

## When to use the terminal during development

The terminal is useful for:

- checking startup flow
- validating runtime status transitions
- diagnosing ROM issues
- confirming quirk presets in effect
- inspecting save/load state behavior
- checking audio initialization success/failure

## Localization of debug logs

Debug strings are not stored in the main UI translation files.

They use:

- `src/debug/i18n/common.json`
- `src/debug/i18n/<lang>.json`

This keeps debug output independent from user-facing UI text.
