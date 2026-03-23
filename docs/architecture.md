# Oxide Architecture

## Overview

```mermaid
flowchart TD

    %% Entry point
    Main[main.rs] --> App[app.rs]

    %% App lifecycle
    App --> Init[Initialize application]
    App --> Loop[Main loop]

    %% Main loop flow
    Loop --> CPU[cpu.rs]
    CPU --> Memory[memory.rs]
    Memory --> Display[display.rs]

    %% Input
    Loop --> Input[gamepad.rs / keypad.rs]
    Input --> CPU

    %% Audio
    Loop --> Audio[audio.rs]

    %% UI
    Loop --> UI[ui/ (top_bar, settings, etc.)]

    %% Localization
    App --> I18n[i18n.rs + JSON]

    %% Feedback loop
    Display --> Loop
