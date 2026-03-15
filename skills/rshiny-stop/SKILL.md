---
name: rshiny-stop
description: "Stop a running R Shiny app started by /rshiny-run"
permissions:
  bash:
    - description: "Read port and kill Shiny process"
      autoApprove: true
      patterns:
        - "cat ~/.claude/rshiny-state/port.txt"
        - "lsof -ti:*"
        - "PID=$(lsof -ti:*"
        - "kill -15 *"
        - "kill -9 *"
        - "sleep *"
---

# Stop R Shiny App

Kill a running Shiny app started by `/rshiny-run` and close the browser tab.

## Steps

1. Read port from `~/.claude/rshiny-state/port.txt`. If file doesn't exist: report "No app tracked. Run `/rshiny-run` first."

2. Check `lsof -ti:<PORT>` for a running process. If nothing found: report "No process on port <PORT>."

3. Graceful shutdown:
   ```bash
   PID=$(lsof -ti:<PORT>) && kill -15 $PID 2>/dev/null; sleep 2
   ```

4. If still alive, force kill:
   ```bash
   PID=$(lsof -ti:<PORT>) && kill -9 $PID 2>/dev/null
   ```

5. Close the browser tab using `browser_tabs` with `action: "close"`. Do NOT use `browser_close` - that closes the entire browser context and may destroy other tabs.

6. Report: "App stopped. Log preserved at `~/.claude/rshiny-state/shiny.log`"
