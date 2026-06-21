TaskTimeTracking — Log timestamped task entries with a short message into [[TimeTrackingLog]].
The top widget shows your most recent entry and how long ago it was started. Below it, buttons let you re-log any past task in one click. Use the Track New Task button (or the TimeTracking command) to add a free-text entry, and the /TimeTrackingButton slash command to insert a reusable quick-log button anywhere in your space.

${topLogEntry()}

${logEntryButtons()}

${widgets.commandButton("⏱ Track New Task", "TimeTracking")}

${widgets.commandButton("↩ Undo Last Entry", "UndoLastTimeTracking")}

${widgets.commandButton("System: Reload")}

```space-lua
local LOG_PAGE = "TimeTrackingLog"
local ICON = "⏱"

local function now()
  return os.date("%Y-%m-%d %H:%M:%S")
end

local function readLog()
  local ok, text = pcall(space.readPage, LOG_PAGE)
  if not ok then
    editor.flashNotification("⚠️ Could not read " .. LOG_PAGE, true)
    return ""
  end
  return text or ""
end

-- Returns (unix_time, msg) or (nil, nil).
local function parseTimestamp(line)
  local y, mo, d, h, mi, s, msg = line:match(
    "^(%d%d%d%d)-(%d%d)-(%d%d) (%d%d):(%d%d):(%d%d) (.+)$"
  )
  if not msg then return nil, nil end
  local t = os.time({
    year = tonumber(y), month = tonumber(mo), day = tonumber(d),
    hour = tonumber(h), min  = tonumber(mi),  sec = tonumber(s)
  })
  return t, msg
end

-- Parses all log lines into an array of { time, msg } (newest first)
local function parseEntries(text)
  local entries = {}
  for line in text:gmatch("[^\n]+") do
    local t, msg = parseTimestamp(line)
    if msg then
      table.insert(entries, { time = t, msg = msg })
    end
  end
  return entries
end

-- Returns cumulative seconds per task name.
-- Each entry "owns" the time from its timestamp until the next (newer) entry.
-- The most recent entry owns time from its timestamp until now.
local function calcCumulative(entries)
  local cumulative = {}
  local nextTime = os.time()
  for _, entry in ipairs(entries) do
    -- entries are newest-first
    local duration = os.difftime(nextTime, entry.time)
    if duration > 0 then
      cumulative[entry.msg] = (cumulative[entry.msg] or 0) + duration
    end
    nextTime = entry.time
  end
  return cumulative
end

-- Pass showSeconds=true for "since X ago" display in topLogEntry;
-- omit it (or pass false) for cumulative button durations.
local function formatDuration(seconds, showSeconds)
  if seconds < 60 then
    if showSeconds then return math.floor(seconds) .. "s" end
    return "< 1m"
  elseif seconds < 3600 then
    return math.floor(seconds / 60) .. "m"
  elseif seconds < 86400 then
    local h = math.floor(seconds / 3600)
    local m = math.floor((seconds % 3600) / 60)
    return string.format("%dh %02dm", h, m)
  else
    return math.floor(seconds / 86400) .. "d"
  end
end

command.define({
  name = "TimeTracking",
  description = "Writes timestamp + message to TimeTrackingLog.md",
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

command.define({
  name = "UndoLastTimeTracking",
  description = "Removes the most recent entry from TimeTrackingLog",
  run = function()
    local text = readLog()
    if text == "" then
      editor.flashNotification("Nothing to undo.")
      return
    end
    local firstLine, rest = text:match("^([^\n]+)\n?(.*)")
    if not firstLine then
      editor.flashNotification("Nothing to undo.")
      return
    end
    local _, msg = parseTimestamp(firstLine)
    local label = msg or firstLine
    local confirm = editor.prompt('Undo "' .. label .. '"? Type YES to confirm:')
    if confirm ~= "YES" then
      editor.flashNotification("Undo cancelled.")
      return
    end
    space.writePage(LOG_PAGE, rest or "")
    editor.flashNotification("↩ Removed: " .. label)
  end
})

slashCommand.define({
  name = "TimeTrackingButton",
  description = "Creates a TimeTracking button with a message",
  run = function()
    local msg = editor.prompt("Button message:")
    if not msg or msg == "" then return end
    local safeMsg = msg:gsub('"', '\\"')
    editor.insertAtCursor(
      '${widgets.commandButton("' .. ICON .. ' ' .. safeMsg .. '", "TimeTracking", "' .. safeMsg .. '")}'
    )
  end
})

function topLogEntry()
  local text = readLog()
  if text == "" then return "_No entries yet in TimeTrackingLog.md_" end
  local firstLine = text:match("^([^\n]+)")
  if not firstLine then return "_No entry found_" end
  local logTime, msg = parseTimestamp(firstLine)
  if not msg then return firstLine end
  local elapsed = os.difftime(os.time(), logTime)
  return "**" .. msg .. "** _(since " .. formatDuration(elapsed, true) .. ")_"
end

function logEntryButtons()
  local text = readLog()
  if text == "" then return "_No entries yet in TimeTrackingLog.md_" end

  local entries    = parseEntries(text)
  local cumulative = calcCumulative(entries)

  local seen           = {}
  local buttonElements = {}

  for _, entry in ipairs(entries) do
    local msg = entry.msg
    if not seen[msg] then
      seen[msg] = true
      local timeStr = formatDuration(cumulative[msg] or 0)
      local row = dom.div {
        style = "display:flex; align-items:center; gap:8px; margin-bottom:4px;",
        widgets.commandButton(ICON .. " " .. msg, "TimeTracking", msg),
        dom.span {
          style = "color:#888; font-size:0.85em; white-space:nowrap;",
          timeStr
        }
      }
      table.insert(buttonElements, row)
    end
  end

  if #buttonElements == 0 then return "_No entries found_" end
  return widget.htmlBlock(dom.div(buttonElements))
end
```
