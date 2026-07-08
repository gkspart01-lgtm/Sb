```space-lua
function getSecret(key)
  local content = space.readPage("secret")
  local fm = index.extractFrontmatter(content)
  local val = fm.frontmatter[key]
  if val == nil then
    return "⚠️ Secret '" .. key .. "' nicht gefunden"
  end
  return val
end

local function yamlEscape(s)
  s = tostring(s)
  s = s:gsub("\\", "\\\\")
  s = s:gsub('"', '\\"')
  return '"' .. s .. '"'
end

command.define {
  name = "Secret: Auswahl als Secret speichern",
  run = function()
    local sel = editor.getSelection()
    if sel.from == sel.to then
      editor.flashNotification("Bitte zuerst den Wert markieren", "error")
      return
    end

    local key = editor.prompt("Name (Key) für das Secret:")
    if not key or key == "" then
      editor.flashNotification("Abgebrochen", "error")
      return
    end

    local value = editor.getText():sub(sel.from + 1, sel.to)

    local pageName = "secret"
    local secrets, rest = {}, ""
    if space.pageExists(pageName) then
      local fm = index.extractFrontmatter(space.readPage(pageName), { removeFrontMatterSection = true })
      secrets = fm.frontmatter or {}
      rest = fm.text
    end

    secrets[key] = value

    local lines = { "---" }
    for k, v in pairs(secrets) do
      table.insert(lines, k .. ": " .. yamlEscape(v))
    end
    table.insert(lines, "---\n")

    space.writePage(pageName, table.concat(lines, "\n") .. rest)

    editor.replaceRange(sel.from, sel.to, "${getSecret('" .. key .. "')}")
    editor.flashNotification("Secret '" .. key .. "' gespeichert")
  end
}
```
