# Oxide — Architecture

Oxide is a CHIP-8 emulator written in Rust, built on top of [egui](https://github.com/emilk/egui) / [eframe](https://github.com/emilk/eframe_template). This document describes the crate layout, module responsibilities, and the key data flows at runtime.

---

## Module Map

```mermaid
graph TD
    main["main.rs\n(entry point)"]
    app["app.rs\nOxide struct\n(app state + eframe::App)"]
    cpu["cpu.rs\nCPU struct\n(fetch / decode / execute)"]
    display["display.rs\nDisplay struct\n(64×32 framebuffer)"]
    keypad["keypad.rs\nKeypad struct\n(16-key state)"]
    memory["memory.rs\n(constants, fontset)"]
    audio["audio.rs\nAudioEngine\n(rodio buzzer)"]
    gamepad["gamepad.rs\n(gilrs controller polling)"]
    fonts["fonts.rs\n(egui font setup)"]
    i18n["i18n.rs\n(UI translations)"]
    debug["debug.rs\n(dev logging)"]
    types["types.rs\n(enums & config types)"]
    utils["utils.rs\n(helpers: key mapping, window size)"]
    constants["constants.rs\n(VERSION, pixel sizes)"]

    ui["ui/"]
    top_bar["ui/top_bar.rs"]
    bottom_bar["ui/bottom_bar.rs"]
    main_panel["ui/main_panel.rs"]
    settings["ui/settings.rs"]
    terminal["ui/debug_terminal.rs"]

    main --> app
    main --> fonts
    app --> cpu
    app --> display
    app --> keypad
    app --> audio
    app --> gamepad
    app --> i18n
    app --> debug
    app --> types
    app --> utils
    app --> constants
    app --> ui
    cpu --> memory
    cpu --> display
    cpu --> keypad
    ui --> top_bar
    ui --> bottom_bar
    ui --> main_panel
    ui --> settings
    ui --> terminal
    top_bar --> app
    bottom_bar --> app
    main_panel --> app
    settings --> app
    terminal --> app
```

---

## Core Data Structures

```mermaid
classDiagram
    class Oxide {
        +CPU cpu
        +Display display
        +Keypad keypad
        +AudioEngine audio_engine
        +CpuQuirks quirks
        +AppTheme theme
        +Langue langue
        +u8 video_scale
        +bool vsync
        +bool fullscreen
        +bool en_pause
        +bool rom_chargee
        +Vec~u8~ rom_data
        +Option~EmuSnapshot~[3] savestates
        +update()
        +save()
        +load_rom_path()
        +reset_rom()
        +stop_emulation()
        +toggle_pause()
        +save_state_slot_manual()
        +load_state_slot()
    }

    class CPU {
        +u8[16] v
        +u16 i
        +u16 pc
        +u8 sp
        +u16[16] stack
        +u8 delay_timer
        +u8 sound_timer
        +u8[4096] memory
        +cycle()
        +tick_timers()
        +load_program()
        +hard_reset()
    }

    class Display {
        +Vec~u8~ pixels
        +clear()
        +get_pixel()
        +set_pixel()
    }

    class Keypad {
        +bool[16] keys
        +is_pressed()
        +first_pressed()
        +set_all()
    }

    class AudioEngine {
        +set_buzzer()
        +stop()
    }

    class EmuSnapshot {
        +CPU cpu
        +Display display
        +Vec~u8~ memory
    }

    class CpuQuirks {
        +bool shift_uses_vy
        +bool jump_uses_vx
        +bool draw_clips
        +bool load_store_increment_i
        +bool logic_clears_vf
    }

    Oxide --> CPU
    Oxide --> Display
    Oxide --> Keypad
    Oxide --> AudioEngine
    Oxide --> CpuQuirks
    Oxide --> EmuSnapshot
    EmuSnapshot --> CPU
    EmuSnapshot --> Display
```

---

## Frame Update Loop

Every frame, `eframe` calls `Oxide::update()`. The sequence is:

```mermaid
sequenceDiagram
    participant eframe
    participant Oxide
    participant CPU
    participant Display
    participant Keypad
    participant AudioEngine

    eframe->>Oxide: update(ctx, frame)
    alt Splash active
        Oxide->>Oxide: render splash screen
        Oxide-->>eframe: return early
    end
    Oxide->>Oxide: handle pending file open
    Oxide->>Oxide: sync viewport / fullscreen state
    Oxide->>Oxide: handle_shortcuts(ctx)
    Oxide->>Keypad: update_keypad_from_input(ctx)
    Oxide->>Oxide: run_emulator_step(dt)
    loop CPU cycles (up to 2000/frame)
        Oxide->>CPU: cycle(display, keypad, quirks)
        CPU->>Display: set_pixel / clear
        CPU->>Keypad: is_pressed / first_pressed
    end
    loop 60 Hz timer ticks
        Oxide->>CPU: tick_timers()
    end
    CPU-->>AudioEngine: sound_timer > 0 → set_buzzer(true)
    Oxide->>Oxide: seed_terminal_boot_logs()
    Oxide->>Oxide: log_config_changes()
    Oxide->>Oxide: update_terminal_log()
    Oxide->>eframe: render top_bar, bottom_bar, main_panel
    alt Settings open
        Oxide->>eframe: render settings viewport
    end
    alt Terminal open
        Oxide->>eframe: render debug_terminal viewport
    end
```

---

## CPU Instruction Cycle

```mermaid
flowchart TD
    A([Start cycle]) --> B[fetch: read 2 bytes at PC\nPC += 2]
    B --> C[decode: split nibbles n1 n2 n3 n4]
    C --> D{n1?}
    D -- 0x0 --> E[CLS / RET]
    D -- 0x1 --> F[JP addr]
    D -- 0x2 --> G[CALL addr]
    D -- 0x3-5-9 --> H[Skip conditionals]
    D -- 0x6-7 --> I[LD / ADD Vx byte]
    D -- 0x8 --> J[ALU ops 0-7 E]
    D -- 0xA --> K[LD I addr]
    D -- 0xB --> L[JP V0+addr]
    D -- 0xC --> M[RND Vx byte]
    D -- 0xD --> N[DRW Vx Vy n\nXOR pixels\nset VF collision]
    D -- 0xE --> O[SKP / SKNP key]
    D -- 0xF --> P[Timers / BCD\nLoad-Store / Font]
    N --> Q[Display::set_pixel]
    J --> R{quirks?}
    R -- shift_uses_vy --> S[source = Vy]
    R -- default --> T[source = Vx]
```

---

## Save State Flow

```mermaid
flowchart LR
    subgraph Save
        A([User triggers save\nslot N]) --> B{Slot already\noccupied?}
        B -- Yes manual --> C[Show confirm dialog]
        C -- Confirmed --> D[save_state_slot_commit]
        B -- No or shortcut --> D
        D --> E[Clone CPU + Display + memory\ninto EmuSnapshot]
        E --> F[Write PersistedSaveState\nto savestates/rom-id/slot-N-timestamp.state]
    end

    subgraph Load
        G([User triggers load\nslot N]) --> H{Snapshot\nexists?}
        H -- No --> I[Show empty slot message]
        H -- Yes --> J[Restore CPU + Display + memory\nfrom EmuSnapshot]
        J --> K[Reset accumulators\nResume emulation]
    end
```

---

## I18n System

```mermaid
flowchart TD
    A[common.json\nshared keys] --> C[parse_translation]
    B[lang/XX.json\ne.g. fr.json] --> C
    C --> D{Valid JSON?}
    D -- Yes --> E[Translations struct\ncached in static Lazy]
    D -- No --> F[Fallback to en.json]
    F --> E
    E --> G[tr - langue - returns static Translations]
    G --> H[UI strings used in\ntop_bar settings terminal]
```

---

## Quirks Presets

| Quirk | CHIP-8 | CHIP-48 | SUPER-CHIP |
|---|---|---|---|
| `shift_uses_vy` | ❌ | ✅ | ✅ |
| `jump_uses_vx` | ❌ | ✅ | ✅ |
| `draw_clips` | ❌ | ❌ | ✅ |
| `load_store_increment_i` | ❌ | ❌ | ❌ |
| `logic_clears_vf` | ❌ | ❌ | ❌ |

---

## Persistence Strategy

`eframe` serializes the `Oxide` struct via `serde` on exit. Fields tagged `#[serde(skip)]` are **not** persisted (runtime-only handles, textures, log buffers). On startup the saved JSON is deserialized and runtime fields are re-initialized.

Save states are stored separately as JSON files under `savestates/<rom_name>-<fnv1a_hash>/`.

```mermaid
flowchart LR
    A[eframe storage\napp_key JSON] -- load --> B[Oxide struct\nsettings only]
    C[savestates/\nrom-id/slot-N.state] -- load_savestates_for_rom --> D[EmuSnapshot\nin memory]
    B --> E[reset_runtime_on_startup]
    E --> F[Ready to run]
    D --> F
```
