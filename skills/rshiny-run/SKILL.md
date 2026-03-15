---
name: rshiny-run
description: "Launch an R Shiny app, open it in a browser via Playwright, and follow free-form instructions while monitoring server logs. Usage: /rshiny-run \"create new project from scratch\""
permissions:
  bash:
    - description: "Kill existing Shiny process by port"
      autoApprove: true
      patterns:
        - "lsof -ti:*"
        - "PID=$(lsof -ti:*"
        - "kill -15 *"
        - "kill -9 *"
        - "sleep *"
    - description: "Start Shiny app in background"
      autoApprove: true
      patterns:
        - "TERM=dumb R --no-save --no-restore -e \"shiny::runApp*"
    - description: "Check app readiness and extract port"
      autoApprove: true
      patterns:
        - "curl -s *localhost*"
        - "for i in *grep*Listening*done"
        - "grep *Listening on*"
    - description: "Manage runtime state files"
      autoApprove: true
      patterns:
        - "mkdir -p ~/.claude/rshiny-state"
        - "> ~/.claude/rshiny-state/shiny.log"
        - "echo * > ~/.claude/rshiny-state/port.txt"
        - "cat ~/.claude/rshiny-state/port.txt"
---

# R Shiny Runner with Log Monitoring

Launch a Shiny app, open it in a browser via Playwright, follow free-form instructions autonomously, and monitor server logs after every action.

**Prerequisites:** Playwright MCP plugin must be installed and configured.

## When to Use

- User asks to run/start/launch a Shiny app and interact with it
- User provides instructions like `/rshiny-run "create new project from scratch"`
- You need to test Shiny UI behavior with live error detection

## Runtime State

All runtime artifacts are stored in `~/.claude/rshiny-state/`:
- `shiny.log` - server log output (created fresh each run)
- `port.txt` - the random port Shiny chose (persists across sessions for `/rshiny-stop` and `/rshiny-status`)

## Workflow

### Step 1: Ensure State Directory Exists

```bash
mkdir -p ~/.claude/rshiny-state
```

### Step 2: Cleanup Previous Run

Read the previous port from `~/.claude/rshiny-state/port.txt` (if it exists) and kill any process on it:

```bash
PORT=$(cat ~/.claude/rshiny-state/port.txt 2>/dev/null)
if [ -n "$PORT" ]; then
  PID=$(lsof -ti:$PORT) && kill -15 $PID 2>/dev/null; sleep 2
  PID=$(lsof -ti:$PORT) && kill -9 $PID 2>/dev/null; sleep 1
fi
```

### Step 3: Launch Shiny App (random port)

Truncate the log file, then start the app in the background. Do NOT specify a port - let Shiny pick a random available one:

```bash
> ~/.claude/rshiny-state/shiny.log
```

Then use the Bash tool with `run_in_background: true`:

```bash
TERM=dumb R --no-save --no-restore -e "shiny::runApp('.', launch.browser = FALSE)" >> ~/.claude/rshiny-state/shiny.log 2>&1
```

- `TERM=dumb` prevents ANSI escape codes at the source.
- Shell redirection works with `run_in_background` because the shell processes `>>` before the tool captures output.

### Step 4: Extract Port from Log + Wait for Readiness

Shiny prints `Listening on http://127.0.0.1:XXXX` when ready. Poll the log for this line (up to 90 seconds for large apps):

```bash
for i in {1..45}; do
  PORT=$(grep -oE 'Listening on http://127\.0\.0\.1:[0-9]+' ~/.claude/rshiny-state/shiny.log 2>/dev/null | grep -oE '[0-9]+$')
  if [ -n "$PORT" ]; then
    echo "$PORT" > ~/.claude/rshiny-state/port.txt
    echo "APP_READY on port $PORT"; exit 0
  fi
  sleep 2
done
echo "APP_TIMEOUT"; exit 1
```

If `APP_TIMEOUT`: read `shiny.log` and report the startup error to the user. Stop.

**Note:** Uses `grep -oE` (extended regex) instead of `grep -oP` (Perl regex) for macOS compatibility.

### Step 5: Open Browser

1. Read the port: `cat ~/.claude/rshiny-state/port.txt`
2. Use `browser_navigate` to `http://localhost:<PORT>`
3. Use `browser_wait_for` with a known UI element (app title or tab) to confirm Shiny has fully initialized
4. Take initial `browser_snapshot` to confirm app loaded

### Step 6: Instruction Loop

If the user provided instructions (e.g., `"create new project from scratch"`), execute an autonomous interaction loop. If no instructions provided, just launch, navigate, snapshot, and report - the app stays running for manual interaction.

The loop runs for up to 100 iterations. After each browser action, wait 1-2 seconds for Shiny reactivity to settle, then check both server logs and browser console for errors. If the same action fails 3 times in a row, stop and report. On hard errors (server crash, disconnect), abort the loop and report what was accomplished.

**Available Playwright actions (full vocabulary):**

- `browser_snapshot` - read current UI state (do this before every decision)
- `browser_click` - click buttons, links, tabs (supports `button: "right"` for context menus)
- `browser_type` / `browser_fill_form` - fill text inputs
- `browser_select_option` - select from dropdowns
- `browser_press_key` - keyboard actions (Enter, Escape, Tab, arrow keys, Ctrl+S)
- `browser_wait_for` - wait for text to appear or disappear (use `textGone` for loading spinners)
- `browser_file_upload` - upload files when encountering file input elements
- `browser_handle_dialog` - dismiss JS alert/confirm/prompt dialogs
- `browser_drag` - drag and drop operations
- `browser_hover` - hover for tooltips or menus
- `browser_take_screenshot` - visual verification when snapshot isn't enough
- `browser_console_messages` - check for client-side JS errors
- `browser_tabs` - manage browser tabs (use `action: "close"` for cleanup)

**Per-iteration flow:**

1. `browser_snapshot` to read current UI state
2. **Disconnect check**: if snapshot shows "Disconnected from server" or "Connection lost", abort and report
3. **Dialog check**: if a JS dialog is blocking, use `browser_handle_dialog` to dismiss it
4. Analyze snapshot against user's instruction - what's done, what's next, is it complete?
5. If complete, break and report
6. Execute the next Playwright action
7. Wait 1-2 seconds for Shiny reactivity to settle (`browser_wait_for` with `time: 1` or `time: 2`)
8. **Check server logs**: read `~/.claude/rshiny-state/shiny.log` from last-read line forward (incremental). Scan new lines for errors:
   - **Hard errors** (abort loop): `"Error in "`, `"Error:"`, `"Traceback"`, `"Shiny application error"`, `"disconnected from server"`, `"fatal"`, `"Execution halted"`
   - **Warnings** (log and continue, mention in final report): `"Warning:"`
   - **Informational** (ignore): `"Listening on"`, deprecation notices
9. **Check browser console**: `browser_console_messages` with `level: "warning"`. Flag `"Uncaught"`, `"TypeError"`, `"WebSocket"` errors.
10. Brief progress note: "Step N: [action taken], [log status]"

**Completion heuristics** - the instruction is likely complete when:

- A success toast/notification appears after a form submission
- The UI returns to a "home" or list view after the requested action
- The expected outcome is visible in the snapshot (e.g., new project appears in list)

### Step 7: Report

- Summary of actions taken (count and key steps)
- Any errors or warnings found in logs
- Final screenshot for visual confirmation
- App status (still running on port from `port.txt`)
- "Use `/rshiny-stop` to shut down, `/rshiny-logs` to view server output"

## Key Behaviors

- **Idempotent**: re-running kills existing app first
- **Log-aware**: every browser action triggers both server log and browser console checks
- **Autonomous**: interprets natural language instructions and decides Playwright actions from snapshots
- **Bounded**: max 100 iterations, 3-failure circuit breaker, abort on hard errors
- **Settle-aware**: waits for Shiny reactivity between actions

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using shell `&` for background | Use Bash tool's `run_in_background: true` parameter |
| Specifying a port in runApp | Let Shiny pick a random port, extract from log |
| Navigating before app is ready | Wait for "Listening on" line in log before proceeding |
| Forgetting `launch.browser = FALSE` | Always set it to prevent duplicate browser windows |
| Using `browser_close` to close tab | Use `browser_tabs` with `action: "close"` instead |
| Skipping log check after actions | Always read shiny.log after every browser action |
| Not waiting for reactivity | Wait 1-2 seconds after each action before next snapshot |
| Hardcoding element refs | Refs change between snapshots; always get fresh snapshot |
| Using `grep -oP` on macOS | Use `grep -oE` with two-stage extraction instead |
