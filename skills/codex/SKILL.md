---
name: codex
description: Use Codex CLI (OpenAI) as a second opinion for code review, brainstorming, and plan review. Triggers on "codex review", "second opinion", "codex brainstorm", "codex plan", or when the user wants an external LLM perspective.
permissions:
  bash:
    - description: "Run codex review on branch changes"
      autoApprove: true
      patterns:
        - "codex review *"
    - description: "Run codex exec for brainstorming or plan review"
      autoApprove: true
      patterns:
        - "codex exec *"
---

# Codex - Second Opinion via OpenAI

Use OpenAI's Codex CLI to get an independent perspective on code changes, brainstorming, or plans.

## Usage

```
/codex review              # Review current branch changes against base
/codex review <base>       # Review against specific base branch
/codex brainstorm <topic>  # Brainstorm a topic or feature
/codex plan <file>         # Review a plan document
```

## Configuration Prompt

**Before running any Codex command, ask the user:**

> Codex defaults: **gpt-5.4**, reasoning effort **xhigh**. Go with defaults or change model/effort?

Wait for the user's response. If they confirm defaults or say nothing specific, use:
- `-c model="gpt-5.4" -c model_reasoning_effort="xhigh"`

If they want changes, apply their preferences to the `-c` flags. Valid models include `o3`, `o4-mini`, `gpt-4.1`, `gpt-5`, `gpt-5.4`, etc. Valid reasoning efforts: `low`, `medium`, `high`, `xhigh`.

Append the chosen `-c` flags to every `codex review` or `codex exec` command.

## Modes

### 1. Code Review (`/codex review [base]`)

Run `codex review` on the current branch's changes.

```bash
# Default: review uncommitted changes
codex review --uncommitted -c model="gpt-5.4" -c model_reasoning_effort="xhigh"

# Against a specific base branch
codex review --base <branch> -c model="gpt-5.4" -c model_reasoning_effort="xhigh"
```

**Workflow:**
1. Determine the base branch: use the argument if provided, otherwise use `--uncommitted` for staged/unstaged changes
2. Run `codex review` with the appropriate flag
3. Present the review output to the user
4. Highlight any findings that conflict with your own assessment

### 2. Brainstorm (`/codex brainstorm <topic>`)

Use `codex exec` to brainstorm approaches for a feature or problem.

```bash
codex exec --full-auto -c model="gpt-5.4" -c model_reasoning_effort="xhigh" -o /tmp/codex-brainstorm.md "Brainstorm approaches for: <topic>. Consider the existing codebase structure. List pros/cons for each approach. Be concise and direct."
```

**Workflow:**
1. Take the topic from the user's argument
2. Run `codex exec` with the brainstorming prompt
3. Read the output file
4. Present Codex's ideas alongside your own perspective
5. Note where you agree/disagree with the suggestions

### 3. Plan Review (`/codex plan <file>`)

Use `codex exec` to review an implementation plan document.

```bash
codex exec --full-auto -c model="gpt-5.4" -c model_reasoning_effort="xhigh" -o /tmp/codex-plan-review.md "Review this implementation plan at <file>. Check for: missing edge cases, architectural concerns, ordering issues, unnecessary complexity, missing steps. Be critical and direct."
```

**Workflow:**
1. Read the plan file to confirm it exists
2. Run `codex exec` with the review prompt referencing the file path
3. Read the output file
4. Present the review to the user
5. Call out any critical issues Codex found that you missed, or disagree with

## Important Notes

- Default config: `gpt-5.4` with `xhigh` reasoning effort - always ask the user before running
- The `--full-auto` flag gives Codex read access to the workspace without prompts
- Output is written to a temp file via `-o` to capture the full response
- Always present both your own assessment and Codex's - the value is in the contrast
- If Codex and you disagree on something, flag it explicitly for the user to decide

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Running without `--full-auto` on exec | Interactive mode won't work in this context |
| Not reading the `-o` output file | The stdout from exec is progress info, not the answer |
| Blindly agreeing with Codex output | The point is a second opinion - compare and contrast |
