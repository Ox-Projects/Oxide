flowchart TD

    Main[main.rs] --> App[app.rs]

    App --> CPU[cpu.rs]
    App --> Memory[memory.rs]
    App --> Display[display.rs]
    App --> Audio[audio.rs]
    
    App --> Input[gamepad.rs / keypad.rs]
    
    App --> UI[ui/]
    
    UI --> MainPanel[main_panel.rs]
    UI --> TopBar[top_bar.rs]
    UI --> BottomBar[bottom_bar.rs]
    UI --> DebugUI[debug_terminal.rs]
    UI --> Settings[settings.rs]

    App --> I18n[i18n.rs + json files]

    Display --> Fonts[fonts.rs]