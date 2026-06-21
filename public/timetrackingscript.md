

${topLogEntry()}
${logEntryButtons()}
${widgets.commandButton("⏱ Track New Time", "logtime")}

${widgets.commandButton("System: Reload")}

[[logtimes]]

```space-lua
local LOG_PAGE = "logtimes"

local function now()
  return os.date("%Y-%m-%d %H:%M:%S")
end

local function readLog()
  local ok, text = pcall(space.readPage, LOG_PAGE)
  return (ok and text) or ""
end

command.define({
  name = "logtime",
  description = "Writes timestamp + message to logtimes.md",
  run = function(msg)
    if not msg or msg == "" then
      msg = editor.prompt("Enter message:")
      if not msg then return end
    end
    local line = now() .. " " .. msg .. "\n"
    space.writePage(LOG_PAGE, line .. readLog())
    editor.flashNotification("✓ Logged: " .. msg)
  end
})

slashCommand.define({
  name = "logtime-button",
  description = "Creates a LogTime button with a message",
  run = function()
    local msg = editor.prompt("Button message:")
    if not msg or msg == "" then return end
    editor.insertAtCursor(
      '${widgets.commandButton("⏱ ' .. msg .. '", "logtime", "' .. msg .. '")}'
    )
  end
})

function topLogEntry()
  local text = readLog()
  if text == "" then return "_No entries yet in logtimes.md_" end

  local firstLine = text:match("^([^\n]+)")
  if not firstLine then return "_No entry found_" end

  local y, mo, d, h, mi, s, msg = firstLine:match(
    "^(%d%d%d%d)-(%d%d)-(%d%d) (%d%d):(%d%d):(%d%d) (.+)$"
  )
  if not msg then return firstLine end

  local logTime = os.time({
    year = tonumber(y), month = tonumber(mo), day = tonumber(d),
    hour = tonumber(h), min  = tonumber(mi), sec = tonumber(s)
  })
  local elapsed = os.difftime(os.time(), logTime)

  local timeAgo
  if elapsed < 60 then
    timeAgo = math.floor(elapsed) .. " sec"
  elseif elapsed < 3600 then
    timeAgo = math.floor(elapsed / 60) .. " min"
  elseif elapsed < 86400 then
    timeAgo = math.floor(elapsed / 3600) .. " hr"
  else
    timeAgo = math.floor(elapsed / 86400) .. " days"
  end

  return "**" .. msg .. "** _(since " .. timeAgo .. ")_"
end

function logEntryButtons()  
  local text = readLog()  
  if text == "" then return "_No entries yet in logtimes.md_" end  
  
  local seen = {}  
  local buttonElements = {}  
  
  for line in text:gmatch("[^\n]+") do  
    local msg = line:match("^%d%d%d%d%-%d%d%-%d%d %d%d:%d%d:%d%d (.+)$")  
    if msg and not seen[msg] then  
      seen[msg] = true  
      table.insert(buttonElements, widgets.commandButton("⏱ " .. msg, "logtime", msg))  
      table.insert(buttonElements, dom.br())  -- Zeilenumbruch nach jedem Button  
    end  
  end  
  
  if #buttonElements == 0 then return "_No entries found_" end  
  
  return widget.htmlBlock(dom.div(buttonElements))  
end
```