# Oxide — Interface Utilisateur & Paramètres

Documentation de l'interface graphique, des fenêtres et du flux des paramètres.

---

## Disposition de la fenêtre principale

```mermaid
block-beta
    columns 1
    TopBar["Barre de menu (30 px)\nJeu | Émulateur | Vidéo | Contrôles | Raccourcis | Debug | Thème ☀/🌙/🌟"]
    Display["Panneau central\nAffichage CHIP-8 (64×32 × scale)\nOverlay pause / messages"]
    BottomBar["Barre de statut (30 px)\nVersion | ROM | État | CPU Hz | FPS | Volume"]
```

---

## Fenêtres secondaires (viewports détachés)

```mermaid
graph LR
    Main["Fenêtre principale\n(viewport racine)"]

    Settings["Paramètres\n(viewport immédiat)\ncentré sur la fenêtre principale"]
    Terminal["Console debug\n(viewport immédiat)\nà droite de la fenêtre principale"]

    Main -->|"fenetre_settings = true"| Settings
    Main -->|"terminal_active = true"| Terminal
    Settings -->|"OK / Cancel / ✕"| Main
    Terminal -->|"✕ ou checkbox"| Main
```

---

## Onglets des paramètres

```mermaid
flowchart LR
    subgraph Tabs["Onglets"]
        T1["Émulateur\nThème, Langue,\nHz CPU"]
        T2["Vidéo\nVSync, Échelle"]
        T3["Audio\nSon, Volume"]
        T4["Contrôles\nMapping 16 touches\nCHIP-8"]
        T5["Raccourcis\nMapping 11 raccourcis\nclaviers"]
        T6["Debug\nTerminal, Quirks\nCHIP-8"]
    end

    subgraph Actions["Boutons bas de page"]
        B1["OK → apply + fermer"]
        B2["Appliquer → apply"]
        B3["Par défaut → reset onglet"]
        B4["Annuler → restaurer snapshot"]
    end
```

---

## Flux des valeurs temporaires (settings)

```mermaid
stateDiagram-v2
    [*] --> Fermé

    Fermé --> Ouvert : Ouverture\n(snapshot des valeurs live)
    Ouvert --> Ouvert : Modification\n(temp_* mis à jour)
    Ouvert --> Live : Appliquer / OK\napply_temp_values()
    Ouvert --> Live : Annuler\nrestore_snapshots()
    Live --> Fermé : Fenêtre fermée
    Live --> [*] : save() au quit

    note right of Ouvert
        temp_theme, temp_langue,
        temp_vsync, temp_video_scale,
        temp_touches, temp_raccourcis,
        temp_quirks, temp_son_active...
    end note
```

---

## Binding des touches (contrôles)

```mermaid
sequenceDiagram
    actor User
    participant UI as Onglet Contrôles
    participant App

    User->>UI: Clic sur bouton d'une touche CHIP-8
    UI->>App: binding_key = Some(index)\nbinding_key_started = Instant::now()\nbinding_key_skip_first_click = true
    UI->>UI: Affiche "..." sur le bouton

    alt Timeout 3 secondes
        UI->>App: binding_key = None
    else Touche clavier détectée
        UI->>App: temp_touches[index] = label
        UI->>App: binding_key = None
    else Clic souris détecté (skip 1er)
        UI->>App: temp_touches[index] = "MouseLeft/Right/..."
        UI->>App: binding_key = None
    end
```

---

## Pipeline d'entrées clavier → CHIP-8

```mermaid
flowchart TD
    KB["Clavier (egui InputState)\ntouches configurées dans temp_touches"]
    Mouse["Souris (egui PointerButton)\nboutons configurés dans temp_touches"]
    GP["Manette (gilrs)\npoll_chip8_keys()"]
    Term["Terminal debug\nterminal_keypad_states[16]"]

    OR["OR logique\nstates[i] = clavier || souris || manette || terminal"]

    Keypad["Keypad.set_all(states)\n→ CPU.cycle() lit is_pressed()"]

    KB --> OR
    Mouse --> OR
    GP --> OR
    Term --> OR
    OR --> Keypad
```

---

## Thèmes disponibles

```mermaid
flowchart LR
    K["🌟 Kiwano\n(défaut)\nRouge bordeaux\négui custom visuals"]
    D["🌙 Dark\négui::Visuals::dark()"]
    L["☀ Light\négui::Visuals::light()"]

    K -->|"clic icône"| D
    D -->|"clic icône"| L
    L -->|"clic icône"| K
```

---

## Console debug — fonctionnalités

```mermaid
graph TD
    subgraph Terminal["Console Oxide (viewport séparé)"]
        Logs["Zone de logs\n(TextEdit multiline\nread-only)"]
        Search["Barre de recherche\n(filtre lignes)"]
        BtnReport["Bouton 'Rapport de test'\nemit_test_report()"]
        BtnExport["Bouton 'Exporter logs'\nrfd::FileDialog → .txt"]
    end

    subgraph Content["Contenu des logs"]
        Boot["Logs de démarrage\n(seed_terminal_boot_logs)"]
        Config["Changements de config\n(log_config_changes)"]
        Status["Messages de statut\n(update_terminal_log)"]
        Report["Rapport de test F9\nPC, registres, quirks..."]
    end

    BtnReport --> Report
    Boot --> Logs
    Config --> Logs
    Status --> Logs
    Report --> Logs
    Search --> |"filtre"| Logs
    BtnExport --> |"écrit fichier"| Disk[("Disque .txt")]

    subgraph Files["Fichiers de log auto"]
        AppLog["logs/app/latest.logs\n→ archivé en .zip au démarrage"]
        EmuLog["logs/emulator/latest.logs\n→ archivé en .zip au démarrage"]
    end

    Logs --> AppLog
    Status --> EmuLog
```