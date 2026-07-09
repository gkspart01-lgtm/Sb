SecretLib — Store and retrieve secrets as YAML frontmatter on a dedicated page, keeping sensitive values out of your regular notes. Select any text and run the **Secret: Save selection as secret** command to move it into the secrets store and replace it inline with a `${getSecret('key')}` reference.

```space-lua
local SECRET_PAGE = "secret"

function getSecret(key)
  local content = space.readPage(SECRET_PAGE)
  local fm = index.extractFrontmatter(content)
  local val = fm.frontmatter[key]
  if val == nil then
    return "⚠️ Secret '" .. key .. "' not found"
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
  name = "Secret: Save selection as secret",
  run = function()
    local sel = editor.getSelection()
    if sel.from == sel.to then
      editor.flashNotification("Please select a value first", "error")
      return
    end

    local key = editor.prompt("Name (key) for the secret:")
    if not key or key == "" then
      editor.flashNotification("Cancelled", "error")
      return
    end

    local value = editor.getText():sub(sel.from + 1, sel.to)

    local secrets, rest = {}, ""
    if space.pageExists(SECRET_PAGE) then
      local fm = index.extractFrontmatter(space.readPage(SECRET_PAGE), { removeFrontMatterSection = true })
      secrets = fm.frontmatter or {}
      rest = fm.text
    end

    secrets[key] = value

    local lines = { "---" }
    for k, v in pairs(secrets) do
      table.insert(lines, k .. ": " .. yamlEscape(v))
    end
    table.insert(lines, "---\n")

    space.writePage(SECRET_PAGE, table.concat(lines, "\n") .. rest)

    editor.replaceRange(sel.from, sel.to, "${getSecret('" .. key .. "')}")
    editor.flashNotification("Secret '" .. key .. "' saved")
  end
}
```