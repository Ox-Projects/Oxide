# Oxide - Development Guide

This document describes how to build, run, and work on Oxide from source.

## Requirements

- Rust toolchain (`cargo`, `rustc`)
- A desktop environment supported by `eframe`
- Audio output device if you want to validate buzzer playback

## Common commands

### Check the project

```bash
cargo check
```

### Run the app

```bash
cargo run
```

### Build a release binary

```bash
cargo build --release
```

## Important runtime notes

### Startup behavior

- `cargo run` launches the splash viewport first
- runtime emulation state is reset on startup
- persisted user preferences are still restored

### Windows single-instance mode

On Windows, Oxide prevents multiple simultaneous instances using a named mutex.

This is compiled in automatically on Windows builds.

### Debug vs release builds

`src/debug.rs` only prints debug lifecycle logs in debug builds.

That means:

- `cargo run` -> debug logs visible
- `cargo build --release` / release binary -> no debug stdout lifecycle logs by default

## Project layout for contributors

Useful directories:

- `src/`: application source
- `src/ui/`: UI modules
- `src/debug/`: debug i18n support
- `src/assets/`: bundled fonts, icons, splash logo
- `docs/`: technical documentation
- `images/`: repository/project visuals
- `logs/`: runtime logs created locally
- `savestates/`: runtime save-state files created locally

## Working on UI

Main UI rendering lives in:

- `src/ui/top_bar.rs`
- `src/ui/bottom_bar.rs`
- `src/ui/main_panel.rs`
- `src/ui/settings.rs`
- `src/ui/debug_terminal.rs`

Theme visuals are centralized in `src/app.rs` through `visuals_for_theme()` and the custom Kiwano visuals setup.

## Working on localization

UI translations:

- `src/i18n/common.json`
- `src/i18n/<lang>.json`

Debug translations:

- `src/debug/i18n/common.json`
- `src/debug/i18n/<lang>.json`

When adding a new shared term that is identical across all languages, prefer `common.json`.

## Working on save states

Save-state logic is centered in `src/app.rs` and documented in `docs/SAVE_STATES.md`.

Important reminder:

- save states are ROM-specific
- UI persistence and save-state persistence are separate systems

## Encoding / mojibake guidance

The repository is configured to reduce encoding issues:

- `.editorconfig`
- `.gitattributes`
- local VS Code UTF-8 settings

Recommended environment:

- UTF-8 editor configuration
- PowerShell 7 or an UTF-8-capable terminal on Windows

## Suggested contribution workflow

1. Create or switch to your working branch.
2. Run `cargo check` before and after changes.
3. Update docs when behavior changes.
4. Keep user-facing strings in i18n files.
5. Keep technical constants in Rust code.
6. Verify that detached viewports still behave correctly.
7. Commit once the code and docs are aligned.
