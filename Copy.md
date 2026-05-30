${command.define {
  name = "My: Copy Full Page to Clipboard",
  description = "Kopiert die komplette Seite als Markdown in die Zwischenablage",
  run = function()
    local text = editor.getText()  -- Holt den gesamten Inhalt
    -- Optional: Bereinigen (z. B. Frontmatter entfernen)
    -- text = string.gsub(text, "^---.-^---%s*", "")
    
    editor.copyToClipboard(text)   -- Oder den Share-Mechanismus nutzen
    editor.flashNotification("✅ Vollständige Seite in die Zwischenablage kopiert!")
  end
}}


