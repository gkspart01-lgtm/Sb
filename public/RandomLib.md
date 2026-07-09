```space-lua

local function generatePassword(length, includeUpper, includeNumbers, includeSpecial)
    length = length or 16
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
    math.randomseed(os.time() + math.random(1000))  -- Bessere Zufälligkeit

    for i = 1, length do
        local rand = math.random(#chars)
        password = password .. string.sub(chars, rand, rand)
    end

    return password
end

-- Custom Command: /password
command.define {
    name = "Generate Password",
    key = "Ctrl-Shift-P",           -- Optional: Tastenkürzel
    run = function()
        -- Standard: 16 Zeichen, mit Großbuchstaben + Zahlen
        local pw = generatePassword(16, true, true, true)
        
        editor.insertAtCursor(pw)
        editor.flashNotification("✅ Passwort generiert und eingefügt! (" .. #pw .. " Zeichen)")
    end
}
```

