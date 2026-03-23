# Oxide — CPU & Émulation CHIP-8

Documentation technique du cœur d'émulation CHIP-8 d'Oxide.

---

## Registres du CPU CHIP-8

```mermaid
classDiagram
    class CPU {
        +v : [u8; 16]       %% V0..VF
        +i : u16            %% Registre index
        +pc : u16           %% Program Counter
        +sp : u8            %% Stack Pointer
        +stack : [u16; 16]  %% Pile d'appels
        +delay_timer : u8   %% Timer délai (60 Hz)
        +sound_timer : u8   %% Timer sonore (60 Hz)
        +memory : [u8; 4096]
        +new() CPU
        +hard_reset()
        +load_program(program) usize
        +cycle(display, keypad, quirks)
        +tick_timers()
    }

    class CpuQuirks {
        +shift_uses_vy : bool
        +jump_uses_vx : bool
        +draw_clips : bool
        +load_store_increment_i : bool
        +logic_clears_vf : bool
    }

    class Display {
        +pixels : Vec~u8~    %% 64×32
        +clear()
        +get_pixel(x, y) bool
        +set_pixel(x, y) bool
    }

    class Keypad {
        +keys : [bool; 16]
        +is_pressed(key) bool
        +first_pressed() Option~u8~
        +set_all(keys)
    }

    CPU --> CpuQuirks : utilise
    CPU --> Display : écrit
    CPU --> Keypad : lit
```

---

## Disposition de la mémoire CHIP-8

```mermaid
block-beta
    columns 1
    block:mem["Mémoire (4096 octets)"]
        A["0x000 – 0x04F\n(réservé)"]
        B["0x050 – 0x09F\nFontset intégré\n(glyphes 0–F, 5 octets chacun)"]
        C["0x0A0 – 0x1FF\n(réservé interpréteur)"]
        D["0x200 – 0xFFF\nProgramme ROM\n(PROGRAM_START)"]
    end
```

---

## Cycle d'exécution CPU

```mermaid
flowchart TD
    A([cycle appelé]) --> B["fetch()\nLit 2 octets à PC\nPC += 2"]
    B --> C["execute(opcode, ...)"]
    C --> D{Nibble haut}

    D --> |0x0| E["CLS / RET"]
    D --> |0x1| F["JP addr"]
    D --> |0x2| G["CALL addr"]
    D --> |0x3/4/5/9| H["Skip conditionnel"]
    D --> |0x6| I["LD Vx, nn"]
    D --> |0x7| J["ADD Vx, nn"]
    D --> |0x8| K["Ops ALU\n(OR, AND, XOR,\nADD, SUB, SHR, SHL)"]
    D --> |0xA| L["LD I, addr"]
    D --> |0xB| M["JP V0+addr / Vx+addr"]
    D --> |0xC| N["RND Vx, nn"]
    D --> |0xD| O["DRW Vx, Vy, n\n(XOR sprite, collision)"]
    D --> |0xE| P["SKP / SKNP\n(touche pressée)"]
    D --> |0xF| Q["LD/ADD/BCD/STR/LD\n(timers, I, mémoire)"]
```

---

## Gestion des timers

```mermaid
sequenceDiagram
    participant App
    participant CPU
    participant Audio

    loop À ~60 Hz (timer_accumulator)
        App->>CPU: tick_timers()
        CPU->>CPU: delay_timer -= 1 (si > 0)
        CPU->>CPU: sound_timer -= 1 (si > 0)
    end

    App->>App: buzzer_active = sound_timer > 0\n&& son_active && volume > 0
    App->>Audio: set_buzzer(buzzer_active, volume)
```

---

## Quirks de compatibilité

| Quirk | CHIP-8 | CHIP-48 | SUPER-CHIP | Effet |
|---|:---:|:---:|:---:|---|
| `shift_uses_vy` | ❌ | ✅ | ✅ | `8xy6/8xyE` décale VY plutôt que VX |
| `jump_uses_vx` | ❌ | ✅ | ✅ | `Bxnn` saute à `Vx + nnn` |
| `draw_clips` | ❌ | ❌ | ✅ | Les sprites sont coupés aux bords |
| `load_store_increment_i` | ❌ | ❌ | ❌ | `Fx55/Fx65` incrémentent I |
| `logic_clears_vf` | ❌ | ❌ | ❌ | Ops logiques remettent VF à 0 |

```mermaid
flowchart LR
    subgraph Presets
        P1["CHIP-8\n(défaut)"]
        P2["CHIP-48"]
        P3["SUPER-CHIP"]
        P4["Custom"]
    end

    P1 -->|"tous quirks\nà false"| Q["CpuQuirks"]
    P2 -->|"shift_vy + jump_vx"| Q
    P3 -->|"shift_vy + jump_vx\n+ draw_clips"| Q
    P4 -->|"configuration\nmanuelle"| Q
```

---

## Rendu de l'affichage CHIP-8

```mermaid
flowchart TD
    FB["Display.pixels\n[u8; 64×32]"]
    Scale["pixel_size calculé\n(video_scale × 10 px\nou plein écran adaptatif)"]
    Painter["egui Painter\nrect_filled() × 2048\nblanc si pixel=1\nnoir si pixel=0"]
    Overlay["Overlay optionnel\n(pause / message statut)"]

    FB --> Scale --> Painter --> Overlay
```