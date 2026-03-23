# Oxide — Save States & Persistance

Documentation du système de sauvegarde d'états et de persistance des données.

---

## Vue d'ensemble

Oxide dispose de deux systèmes de persistance distincts :

1. **Paramètres utilisateur** — via `eframe::Storage` (JSON interne eframe)
2. **Save states** — fichiers `.state` par ROM, dans le dossier `savestates/`

---

## Structure des données d'un save state

```mermaid
classDiagram
    class PersistedSaveState {
        +version : u32
        +rom_name : String
        +rom_hash : String
        +rom_bytes : Vec~u8~
        +rom_path : String
        +slot : usize
        +meta : SaveStateMeta
        +snapshot : EmuSnapshot
    }

    class SaveStateMeta {
        +name : String
        +timestamp : String
    }

    class EmuSnapshot {
        +cpu : CPU
        +display : Display
        +memory : Vec~u8~
    }

    PersistedSaveState --> SaveStateMeta
    PersistedSaveState --> EmuSnapshot
    EmuSnapshot --> CPU
    EmuSnapshot --> Display
```

---

## Organisation des fichiers sur disque

```
savestates/
└── <nom_rom>-<hash_fnv1a>/          ← un dossier par ROM
    ├── <nom>_01_<timestamp>.state   ← slot 1
    ├── <nom>_02_<timestamp>.state   ← slot 2
    └── <nom>_03_<timestamp>.state   ← slot 3
```

> Un seul fichier `.state` par slot est conservé. L'ancien est supprimé avant chaque écriture.

---

## Identification des ROMs

```mermaid
flowchart LR
    ROM["Données ROM (octets)"]
    FNV["FNV-1a 64-bit\nrom_hash_hex()"]
    Name["Nom de fichier\nsanitize_filename()"]
    Dir["savestates/\n&lt;nom&gt;-&lt;hash&gt;/"]

    ROM --> FNV --> Dir
    Name --> Dir
```

---

## Flux de sauvegarde d'un slot

```mermaid
sequenceDiagram
    actor User
    participant App
    participant Disk

    User->>App: save_state_slot_manual(slot)\nou raccourci clavier

    alt Slot déjà occupé (manual)
        App->>User: Fenêtre de confirmation\n"Écraser le slot N ?"
        User->>App: Confirme
    end

    App->>App: save_state_slot_commit(slot, manual)
    App->>App: savestates[slot] = EmuSnapshot { cpu, display, memory }
    App->>App: savestate_meta[slot] = SaveStateMeta { name, timestamp }
    App->>App: persist_savestate(slot)
    App->>Disk: Supprime ancien fichier du slot
    App->>Disk: Écrit &lt;nom&gt;_&lt;slot&gt;_&lt;timestamp&gt;.state (JSON)
    App->>User: Overlay "État sauvegardé dans le slot N"
```

---

## Flux de chargement d'un slot

```mermaid
flowchart TD
    A([load_state_slot N]) --> B{savestates[N] existe ?}
    B -- Non --> C["status: Slot vide N\noverlay affiché"]
    B -- Oui --> D["cpu = snapshot.cpu\ndisplay = snapshot.display"]
    D --> E{memory.len() correct ?}
    E -- Oui --> F["cpu.memory.copy_from_slice(&snapshot.memory)"]
    E -- Non --> G["cpu.load_program(&rom_data) — fallback"]
    F --> H["rom_chargee = true\nen_pause = false\nreset_runtime_clocks()"]
    G --> H
    H --> I([Émulation reprend])
```

---

## Chargement auto des saves au démarrage d'une ROM

```mermaid
flowchart TD
    ROM([ROM chargée]) --> DIR["savestate_dir_for_rom()\n→ savestates/&lt;nom&gt;-&lt;hash&gt;/"]
    DIR --> HASH["rom_hash_hex() calculé"]

    HASH --> SLOT0["latest_savestate_file(dir, 0, nom)"]
    HASH --> SLOT1["latest_savestate_file(dir, 1, nom)"]
    HASH --> SLOT2["latest_savestate_file(dir, 2, nom)"]

    SLOT0 --> V0{hash + slot\ncorrespondent ?}
    SLOT1 --> V1{hash + slot\ncorrespondent ?}
    SLOT2 --> V2{hash + slot\ncorrespondent ?}

    V0 -- Oui --> L0["savestates[0] = Some(snapshot)\nsavestate_meta[0] = Some(meta)"]
    V1 -- Oui --> L1["savestates[1] = Some(snapshot)"]
    V2 -- Oui --> L2["savestates[2] = Some(snapshot)"]

    V0 -- Non --> N0["savestates[0] = None"]
    V1 -- Non --> N1["savestates[1] = None"]
    V2 -- Non --> N2["savestates[2] = None"]
```

---

## Persistance des paramètres (eframe::Storage)

```mermaid
flowchart LR
    subgraph Sauvegardé ["Persisté (eframe storage)"]
        A["theme, langue, vsync\nvideo_scale, touches\nraccourcis, cycles_par_seconde\nson_active, sound_volume\nquirks, quirks_preset\nterminal_active\nlast_rom_path"]
    end

    subgraph Réinitialisé ["Réinitialisé au démarrage"]
        B["cpu, display, keypad\nrom_data, rom_path\nrom_chargee, en_pause\nsavestates, savestate_meta\nterminal_logs, audio_engine\noverlay, focus flags..."]
    end

    App["Oxide::save()"] --> Sauvegardé
    App -->|"reset_runtime_on_startup()"| Réinitialisé
```

---

## Logs automatiques (rotation ZIP)

```mermaid
flowchart TD
    Start([Démarrage]) --> Check{latest.logs\nexiste ?}
    Check -- Oui --> Zip["Crée logs-app-&lt;timestamp&gt;.zip\ncontenant latest.logs"]
    Zip --> Del["Supprime latest.logs"]
    Del --> New["Ouvre nouveau latest.logs"]
    Check -- Non --> New
    New --> Session["Écriture des logs de session"]
```

```
logs/
├── app/
│   ├── latest.logs              ← session en cours
│   └── logs-app-2025-01-01_...zip
└── emulator/
    ├── latest.logs
    └── logs-emulator-2025-01-01_...zip
```