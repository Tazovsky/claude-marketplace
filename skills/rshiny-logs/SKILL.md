---
name: rshiny-logs
description: "Show recent R Shiny app logs from /rshiny-run"
---

# View Shiny App Logs

Show recent server log output from a Shiny app started by `/rshiny-run`.

No bash permissions needed - uses the Read tool only. ANSI codes are prevented at source (`TERM=dumb` in the launch command).

## Steps

1. Check if `~/.claude/rshiny-state/shiny.log` exists. If not: report "No log file found. Run `/rshiny-run` first."

2. Read the last 200 lines of the log file using the Read tool with offset.

3. Report the log content, highlighting any errors or warnings found:
   - **Errors**: lines containing `"Error in "`, `"Error:"`, `"Traceback"`, `"Shiny application error"`, `"fatal"`, `"Execution halted"`
   - **Warnings**: lines containing `"Warning:"`
