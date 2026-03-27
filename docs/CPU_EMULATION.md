# Oxide - CPU Emulation

This document focuses on the current CHIP-8 emulation core implemented in `src/cpu.rs`.

## CPU state

The emulator models the standard CHIP-8 machine state:

- `V0..VF`: 16 general-purpose 8-bit registers
- `I`: 16-bit index register
- `PC`: program counter
- `SP`: stack pointer
- `stack[16]`: call stack
- `delay_timer`: decremented at 60 Hz
- `sound_timer`: decremented at 60 Hz
- `memory[4096]`: full CHIP-8 memory space

## Memory layout

Current layout used by Oxide:

- `0x000-0x04F`: reserved
- `0x050-0x09F`: built-in fontset (glyphs `0-F`)
- `0x0A0-0x1FF`: reserved / interpreter space
- `0x200-0xFFF`: loaded ROM program area

ROM loading starts at the standard CHIP-8 start address.

## Execution cycle

The CPU cycle follows the standard pattern:

1. Fetch 2 bytes at `PC`
2. Increment `PC`
3. Decode opcode fields
4. Execute the corresponding instruction
5. Mutate CPU, display, keypad, or memory state

The main frame loop decides how many CPU cycles run per frame based on `cycles_par_seconde` and accumulated frame time.

## Implemented opcode groups

Oxide implements the full CHIP-8 opcode set currently targeted by the project, including:

- screen control and return
- jumps and calls
- conditional skips
- register load / immediate add
- register ALU operations
- index register operations
- random masked values
- sprite drawing
- keypad-dependent skips
- timer operations
- BCD conversion
- register-memory transfer operations

## Display drawing behavior

Sprites are drawn through XOR semantics on the logical `64x32` framebuffer.

During `DRW`:

- bits are read from memory at `I`
- pixels are XORed into `Display`
- `VF` is used as the collision flag

Depending on the selected quirk preset, draw behavior can wrap or clip at the display edges.

## Timers

`delay_timer` and `sound_timer` are not tied to raw CPU cycle count.

Instead:

- the app accumulates time separately
- timers are ticked at approximately `60 Hz`
- `sound_timer > 0` drives the audio buzzer

This keeps timers stable even if CPU speed changes.

## Compatibility quirks

Oxide exposes 5 configurable CHIP-8 quirks through `CpuQuirks`:

- `shift_uses_vy`
- `jump_uses_vx`
- `draw_clips`
- `load_store_increment_i`
- `logic_clears_vf`

Available presets:

- `CHIP-8`
- `CHIP-48`
- `SUPER-CHIP`
- `Custom`

Preset mapping is defined in `src/types.rs`.

## Input path

The CPU reads key state from `Keypad`.

`Keypad` itself is populated from a merged input pipeline:

- keyboard
- mouse button bindings
- gamepad state
- debug terminal virtual keypad state

This means the CPU always consumes a single normalized keypad view.

## ROM loading and reset behavior

When a ROM is loaded:

- CPU memory is reset appropriately
- the ROM is copied into program memory
- display and runtime accumulators are reset as needed
- save states associated with the ROM are reloaded from disk if available

A stop/reset action clears active runtime state without wiping persisted user settings.

## Audio coupling

The CPU itself does not own audio playback.

Instead:

- `sound_timer` is decremented in the runtime loop
- `app.rs` checks whether sound should be active
- `AudioEngine` starts or stops the buzzer accordingly

## Practical debugging support

The emulator exposes a test-report path for diagnostics.

`emit_test_report()` can log:

- ROM name
- running/paused state
- current `PC` and opcode
- register dump
- display activity summary
- keypad state
- active quirk preset and flags

This is especially useful when validating ROM behavior or quirk regressions.
