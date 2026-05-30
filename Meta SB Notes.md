Bestrebungen und Kompatibilität
Migration von Obsidian zu SilverBullet: Es gibt konkrete Tools und Skripte dafür. Ein bekanntes GitHub-Projekt (SONDLecT/Obsidian-to-Silverbullet-Conversion-Scripts) bietet Python-Skripte, die Obsidian-spezifische Features anpassen:
Wikilinks mit vollem Pfad (Obsidian ist flacher).
Bild-Embeds und ![[Embeds]].
Callouts/Admonitions.
Entfernen von Dataview-Queries und problematischem YAML-Frontmatter.
Dateinamen-Bereinigung.d3c716
Diese Skripte machen einen Obsidian-Vault „SilverBullet-friendly“. Viele Nutzer berichten in Foren und auf HN davon, dass sie damit erfolgreich migrieren. https://community.silverbullet.md/t/an-admittedly-hacky-approach-to-making-your-obsidian-vault-silverbullet-friendly/167
Gemeinsame Nutzung: Da beide Tools auf reinen Markdown-Dateien basieren, kannst du denselben Ordner/Vault nutzen (z. B. per Docker-Mount). Es gibt Diskussionen zu Hybrid-Setups mit Obsidian, Logseq und Silverbullet. https://community.silverbullet.md/t/utilizing-silverbullet-with-other-markdown-editors/61