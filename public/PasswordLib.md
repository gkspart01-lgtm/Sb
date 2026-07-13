PasswordLib — Generate random passwords with customizable length and character sets, insertable directly into your notes via command palette or the `/password` slash command.

```space-lua
local DEFAULT_LENGTH = 16
local DEFAULT_UPPER = true
local DEFAULT_NUMBERS = true
local DEFAULT_SPECIAL = true

local function generatePassword(length, includeUpper, includeNumbers, includeSpecial)
    length = length or DEFAULT_LENGTH
    includeUpper = includeUpper ~= false
    includeNumbers = includeNumbers ~= false
    includeSpecial = includeSpecial or false

    local lower = "abcdefghijklmnopqrstuvwxyz"
    local upper = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    local numbers = "0123456789"
    local special = "!@#$%^&*()_+-=[]{}|;:,.<>/?"

    local chars = lower
    if includeUpper then chars = chars .. upper end
    if includeNumbers then chars = chars .. numbers end
    if includeSpecial then chars = chars .. special end

    local password = ""
    math.randomseed(os.time() + math.random(1000))  -- better randomness

    for i = 1, length do
        local rand = math.random(#chars)
        password = password .. string.sub(chars, rand, rand)
    end

    return password
end

-- Command palette entry
command.define {
    name = "Password: Generate",
    run = function()
        local pw = generatePassword(DEFAULT_LENGTH, DEFAULT_UPPER, DEFAULT_NUMBERS, DEFAULT_SPECIAL)
        editor.insertAtCursor(pw)
        editor.flashNotification("✅ Password generated and inserted! (" .. #pw .. " characters)")
    end
}

-- Slash command: type /password while editing a note
slashCommand.define {
    name = "password",
    description = "Insert a randomly generated password",
    run = function()
        local pw = generatePassword(DEFAULT_LENGTH, DEFAULT_UPPER, DEFAULT_NUMBERS, DEFAULT_SPECIAL)
        editor.insertAtCursor(pw)
        editor.flashNotification("✅ Password generated and inserted! (" .. #pw .. " characters)")
    end
}
```
