---
name: rshiny-status
description: "Check if an R Shiny app is running (started by /rshiny-run)"
permissions:
  bash:
    - description: "Read port and check app health"
      autoApprove: true
      patterns:
        - "cat ~/.claude/rshiny-state/port.txt"
        - "lsof -ti:*"
        - "curl -s *localhost*"
---

# Check Shiny App Status

Check if a Shiny app started by `/rshiny-run` is still running and healthy.

## Steps

1. Read port from `~/.claude/rshiny-state/port.txt`. If file doesn't exist: report "No app tracked."

2. Check `lsof -ti:<PORT>` for a running process.

3. If no process found: report "Not running (port <PORT> from last run)."

4. If process found, check health:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" http://localhost:<PORT>
   ```
   - **200**: "Running and healthy on port <PORT>"
   - **Other**: "Process on port <PORT> but not responding (HTTP <code>)"
