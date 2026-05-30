# SilverBullet Cheat Sheet

Dieses Cheat Sheet ist für den schnellen Überblick gedacht. Speziell mit Fokus auf Mobile + häufige Workflows.

## Wichtige Meta- & Config-Seiten

- **[[!silverbullet.md/Keyboard Shortcuts]]** → Alle aktuellen Tastenkürzel (inkl. deiner Plugs)
- **[[SETTINGS]]** oder **[[CONFIG]]** → Globale Einstellungen (v2)
- **[[PLUGS]]** → Installierte Plugs verwalten
- **[[index]]** → Startseite / Home
- **[[Library]]** → Offizielle Bibliothek & Templates
- **[[Commands]]** → Alle verfügbaren Befehle

Tipp: Erstelle dir eine persönliche **[[Meta]]**-Seite mit Links zu allem.

## Quick Note (Schnellnotiz)

**Tastenkürzel:** `Alt + Shift + N` (bzw. `Option + Shift + N` auf Mac)

**Alternative:**
- Command Palette öffnen (`Cmd/Ctrl + /`) → „Quick Note“ suchen
- Button in der Toolbar (falls aktiviert)
- Über Template: `Ctrl + Q` + `Q` (manchmal konfiguriert)

Quick Notes landen standardmäßig mit Datum/Zeit + Emoji. Du kannst das Template in `Library/Std/Page Templates/Quick Note` anpassen.

## Hilfreiche Plugs (empfohlen)

Füge diese in deiner **CONFIG**-Seite unter `plugs:` hinzu und führe `Plugs: Update` aus.

- **TreeView** — Baum-Navigation links (sehr nützlich)
- **GraphView** — Graph-Ansicht wie in Obsidian
- **AI** — Integration von lokalen oder Cloud-KI-Modellen
- **Markmap** — Mindmaps aus Markdown
- **PDF** — PDFs direkt ansehen + annotieren
- **Pomodoro** — Timer für Aufgaben
- **PlantUML** / **Excalidraw** — Diagramme & Zeichnungen
- **Git** — Versionskontrolle

Community-Plugs findest du im Forum: [community.silverbullet.md/c/plugs-libraries](https://community.silverbullet.md/c/plugs-libraries/14)

## Tastenkürzel (Auswahl)

| Aktion                    | Shortcut                  |
|---------------------------|---------------------------|
| Command Palette           | `Cmd/Ctrl + /`            |
| Quick Note                | `Alt + Shift + N`         |
| Neue Seite                | `Cmd/Ctrl + K`            |
| Suche im Space            | `Cmd/Ctrl + Shift + F`    |
| Fett                      | `Cmd/Ctrl + B`            |
| Kursiv                    | `Cmd/Ctrl + I`            |
| Heading (Slash)           | `/` dann Text             |
| Outline: Einrücken        | `Tab` oder `Mod + .`      |
| Outline: Ausrücken        | `Shift + Tab`             |

Vollständige Liste immer auf `[[!silverbullet.md/Keyboard Shortcuts]]` anschauen.

## Mobile / Touch Tips & Tricks

- **PWA installieren** → Browser-Menü → „Zum Home-Bildschirm hinzufügen“ (fühlt sich wie eine App an)
- **Command Palette** ist dein bester Freund auf Mobile (`Cmd/Ctrl + /` oder über das Menü)
- **Toolbar** oben zeigt häufige Aktionen (kann mit Custom CSS optimiert werden)
- **2-Finger-Tap / 3-Finger-Tap** — Je nach Browser/PWA können Gesten für Undo, Redo oder Kontextmenü funktionieren. Teste es aus!
- **Virtuelles Keyboard** — Viele nutzen externe Bluetooth-Tastaturen für bessere Shortcuts
- Listen einrücken auf Touch: Lange drücken + Toolbar-Buttons nutzen

**Pro-Tipp Mobile:** Erstelle eine Seite mit häufig genutzten Slash-Commands und verlinke sie in der Sidebar.

## Weitere Tipps & Tricks

- **Slash Commands** (`/`) für fast alles: Templates, Queries, Embeds
- **Queries** sind extrem mächtig: `page where tags = "task" ...`
- **Templates** nutzen: Erstelle eigene mit `#template`-Tag
- **Federation**: Du kannst Seiten aus anderen SilverBullet-Spaces einbinden (z. B. `[[!silverbullet.md/...]]`)
- **Live Preview** vs. Source Mode umschalten
- Regelmäßig `Plugs: Update` und `Index: Reindex` laufen lassen
- Backlinks & Erwähnungen automatisch anzeigen lassen

## Eigene Shortcuts definieren

In der **CONFIG**-Seite möglich. Beispiel:

```yaml
shortcuts:
  - command: "Quick Note"
    key: "Alt-n"