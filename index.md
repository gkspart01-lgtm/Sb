 # useful links
   
  -  [Manual](https://silverbullet.md/Manual)
  -  [[CheatSheet]]
  -  Ghuh

# buttons
${widgets.commandButton("Pomodoro Clock","Floating: Pomodoro Clock")}

${shell.run("git", {"diff", "--exit-code"})}

${query[[from c=command]]}

${query[[
from c = index.tag "command"
order by c.name
]]}



# Recent quick notes
*Create:* ${widgets.commandButton "Quick Note"}

${some(query[[
  from p = tags.page
  where p.name:startsWith("Inbox/")
  order by p.lastModified desc
  limit 10 select templates.fullPageItem(p)
]]) or "_No quick notes yet!_"}

# Recent incomplete tasks
${some(query[[
  from t = tags.task
  where not t.done
  order by t.pageLastModified
  desc limit 10
  select templates.taskItem(t)
]]) or "_All tasks done!_"}

# Recently modified pages
${query[[
  from p = index.contentPages()
  order by p.lastModified desc
  limit 10
  select templates.fullPageItem(p) 
]]}
