# Oxide

A modern CHIP-8 emulator written in Rust with an `egui` / `eframe` interface.

Oxide focuses on a clean desktop UI, configurable emulation behavior, multilingual support, save states, debug tooling, and a polished startup experience.

## Highlights

- Full CHIP-8 CPU implementation with configurable compatibility quirks
- Animated splash screen with bundled logo and version display
- Three application themes: Kiwano, Dark, Light
- 12 UI languages: FR, EN, ES, IT, DE, PT, RU, ZH, JA, KO, AR, HI
- Save/load states with per-ROM slots and metadata
- Debug terminal with live logs, search, export, and test reports
- Keyboard, mouse, and gamepad input support
- Audio buzzer with volume control
- Detached Settings and Debug Terminal windows
- Windows single-instance protection

## Platform support

Oxide is designed to run on:

- Windows
- Linux
- macOS

Windows currently has the most platform-specific integration, including the app icon resource and single-instance guard.

## Getting started

### Run from source

```bash
cargo run
```

### Build a release binary

```bash
cargo build --release
```

### Load a ROM

From the UI:

`Game -> Load game`

Supported ROM extensions:

- `.ch8`
- `.rom`
- `.bin`

## Main features

### Emulation

- 35 CHIP-8 opcodes implemented
- 64x32 framebuffer rendering
- Adjustable video scale from `1x` to `5x`
- VSync toggle and fullscreen support
- Quirk presets: CHIP-8, CHIP-48, SUPER-CHIP, Custom

### Input

- Fully configurable 16-key CHIP-8 mapping
- Configurable keyboard shortcuts
- Mouse button bindings supported in control mapping
- Gamepad polling through `gilrs`

### Audio

- CHIP-8 buzzer driven by `rodio`
- Volume control from `0` to `100`
- Runtime enable/disable toggle

### Save states

- 3 slots per ROM
- Save/load from menu, settings, or shortcuts
- Metadata saved with timestamp and ROM identity
- `.state` files stored under `savestates/`

### Debugging

- Detached debug terminal window
- Runtime logs with search filter
- Export logs to text file
- On-demand test report with CPU/video/quirk snapshot
- Rotating `logs/app` and `logs/emulator` archives

### UI and UX

- Startup splash screen
- Settings window with tabbed configuration flow
- Theme switching from top bar and settings
- Pending-change detection in settings
- Status bar with runtime information

## Default shortcuts

The exact shortcuts are configurable in the UI. Default bindings are defined in `src/types.rs`.

Common defaults:

- `O`: load game
- `P`: pause / resume
- `R`: reset ROM
- `Esc`: stop emulation
- `F11`: fullscreen
- `F1-F3`: save state slots 1-3
- `F5-F7`: load state slots 1-3

## Project documentation

- [ROADMAP.md](ROADMAP.md)
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- [docs/CPU_EMULATION.md](docs/CPU_EMULATION.md)
- [docs/SAVE_STATES.md](docs/SAVE_STATES.md)
- [docs/UI_SETTINGS.md](docs/UI_SETTINGS.md)
- [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md)
- [docs/DEBUG_LOGGING.md](docs/DEBUG_LOGGING.md)

## Repository assets

Project visuals are stored under:

- `images/`
- `src/assets/icons/`
- `src/assets/logo/`
- `src/assets/fonts/`

## Built with

- [Rust](https://www.rust-lang.org/)
- [egui / eframe](https://github.com/emilk/egui)
- [rodio](https://github.com/RustAudio/rodio)
- [gilrs](https://gitlab.com/gilrs-project/gilrs)
- [rfd](https://github.com/PolyMeilex/rfd)
- [image](https://crates.io/crates/image)
- [zip](https://crates.io/crates/zip)
- [chrono](https://crates.io/crates/chrono)

## License

[MIT](LICENSE)
