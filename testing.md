${virtualPage.define {
  pattern = "greeting:(.+)",
  run = function(name)
    return "# Hello, " .. name .. "!\nWelcome to this virtual page."
  end
}}

[[greeting:f]]

```excalidraw
url:testU.excalidraw
height:500px
```