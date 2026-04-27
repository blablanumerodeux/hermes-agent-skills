---
name: hermes-interactive-setup
description: Workflow for providing input to Hermes interactive setup wizards that require numeric menu selection. Direct numeric input fails; must pipe input via echo.
version: 1.1.0
author: Hermes Agent
tags: [hermes, setup, configuration, interactive, piping]
---

# Hermes Interactive Setup Input Workflow

## Problem
When running `hermes gateway setup` or other Hermes interactive setup commands, sending numeric input directly (e.g., typing "1" and pressing Enter) results in:
```
/bin/bash: line 3: 1: command not found
```

This happens because the setup wizard runs in a context where direct input is interpreted as a shell command rather than menu selection.

## Solution
Pipe numeric input to the setup command using `echo`:
```bash
echo "<selection_number>" | hermes gateway setup
```

## Important Note
You must pipe input to the SAME invocation of the setup command. Piping to a new instance (like `echo "7" | hermes gateway setup` while another instance is already running) will start a fresh setup wizard rather than sending input to the running one.

## Step-by-Step Guide

### 1. Start the setup wizard IN YOUR CURRENT TERMINAL
```bash
hermes gateway setup
```

### 2. For each menu prompt, pipe your selection TO THE RUNNING INSTANCE
Instead of typing the number directly in the running wizard, use:
```bash
echo "<number>" | hermes gateway setup
```

### 3. Example workflow (in same terminal session)
```bash
# Start setup (leave this running)
hermes gateway setup

# NOW, in the SAME TERMINAL, pipe your selection:
# When prompted: "Select a platform to configure:"
echo "1" | hermes gateway setup  # Selects Telegram (option 1)

# Continue with platform-specific setup...
# When done configuring platforms:
echo "16" | hermes gateway setup  # Selects "Done" (option 16)

# Setup will complete and offer to install gateway service
```

### 4. Alternative: Provide all inputs at once to a single invocation
You can also pipe multiple inputs sequentially to a single setup invocation:
```bash
{
  echo "1"      # Select Telegram
  echo "token"  # Enter bot token when prompted
  echo "123"    # Enter user ID when prompted
  echo "16"     # Select Done
} | hermes gateway setup
```

## Why This Works
The setup wizard reads from standard input. Piping via `echo` ensures the input is delivered as data rather than being interpreted as a shell command by the invoking shell.

## Applies To
- `hermes gateway setup`
- `hermes setup` (model, terminal, tools, agent sections)
- `hermes profile create` (when interactive)
- Any other Hermes interactive setup/menu commands

## Verification
After completing setup, verify configuration with:
```bash
hermes gateway status
hermes config
```

## Troubleshooting
- If you see "GetPassWarning: Can not control echo on the terminal" - this is normal for password/token inputs
- If setup exits unexpectedly, check that you're sending the correct number of inputs for all prompts
- Use `Ctrl+C` to exit the setup wizard at any time
- REMEMBER: Pipe to the same invocation - don't start new instances with each echo