TaskTimeTracking — Log timestamped task entries with a short message into [[TimeTrackingLog]].
The top widget shows your most recent entry and how long ago it was. Below it, deduplicated buttons let you re-log any past message in one click. Use the Track New Time button (or the TimeTracking command) to add a free-text entry, and the /TimeTrackingButton slash command to insert a reusable quick-log button anywhere in your space.

${topLogEntry()}

${logEntryButtons()}

${widgets.commandButton("⏱ Track New Task", "TimeTracking")}

${widgets.commandButton("System: Reload")}

```space-lua
local LOG_PAGE = "TimeTrackingLog"

local function now()
  return os.date("%Y-%m-%d %H:%M:%S")
end

local function readLog()
  local ok, text = pcall(space.readPage, LOG_PAGE)
  return (ok and text) or ""
end

-- Parses all log lines into an array of { time, msg } (newest first)
local function parseEntries(text)
  local entries = {}
  for line in text:gmatch("[^\n]+") do
    local y, mo, d, h, mi, s, msg = line:match(
      "^(%d%d%d%d)-(%d%d)-(%d%d) (%d%d):(%d%d):(%d%d) (.+)$"
    )
    if msg then
      local t = os.time({
        year = tonumber(y), month = tonumber(mo), day = tonumber(d),
        hour = tonumber(h), min  = tonumber(mi),  sec = tonumber(s)
      })
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
  for _, entry in ipairs(entries) do          -- entries are newest-first
    local duration = os.difftime(nextTime, entry.time)
    if duration > 0 then
      cumulative[entry.msg] = (cumulative[entry.msg] or 0) + duration
    end
    nextTime = entry.time
  end
  return cumulative
end

-- Formats a duration in seconds to a readable string
local function formatDuration(seconds)
  if seconds < 60 then return "< 1m" end
  local h = math.floor(seconds / 3600)
  local m = math.floor((seconds % 3600) / 60)
  if h > 0 then
    return string.format("%dh %02dm", h, m)
  else
    return string.format("%dm", m)
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

slashCommand.define({
  name = "TimeTrackingButton",
  description = "Creates a TimeTracking button with a message",
  run = function()
    local msg = editor.prompt("Button message:")
    if not msg or msg == "" then return end
    editor.insertAtCursor(
      '${widgets.commandButton("⏱ ' .. msg .. '", "TimeTracking", "' .. msg .. '")}'
    )
  end
})

function topLogEntry()
  local text = readLog()
  if text == "" then return "_No entries yet in TimeTrackingLog.md_" end
  local firstLine = text:match("^([^\n]+)")
  if not firstLine then return "_No entry found_" end
  local y, mo, d, h, mi, s, msg = firstLine:match(
    "^(%d%d%d%d)-(%d%d)-(%d%d) (%d%d):(%d%d):(%d%d) (.+)$"
  )
  if not msg then return firstLine end
  local logTime = os.time({
    year = tonumber(y), month = tonumber(mo), day = tonumber(d),
    hour = tonumber(h), min = tonumber(mi), sec = tonumber(s)
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


```
