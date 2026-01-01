# Plannotator Claude Code Plugin

This directory contains the Claude Code plugin configuration for Plannotator.

## Prerequisites

Install the `plannotator` command so Claude Code can use it:

**macOS / Linux / WSL:**
```bash
curl -fsSL https://plannotator.ai/install.sh | bash
```

**Windows PowerShell:**
```powershell
irm https://plannotator.ai/install.ps1 | iex
```

**Windows CMD:**
```cmd
curl -fsSL https://plannotator.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

---

[Plugin Installation](#plugin-installation) · [Manual Installation (Hooks)](#manual-installation-hooks)

---

## Plugin Installation

In Claude Code:

```
/plugin marketplace add backnotprop/plannotator
/plugin install plannotator@plannotator
```

**Important:** Restart Claude Code after installing the plugin for the hooks to take effect.

## Manual Installation (Hooks)

If you prefer not to use the plugin system, add this to your `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          {
            "type": "command",
            "command": "plannotator",
            "timeout": 1800
          }
        ]
      }
    ]
  }
}
```

## How It Works

When Claude Code calls `ExitPlanMode`, this hook intercepts and:

1. Opens Plannotator UI in your browser
2. Lets you annotate the plan visually
3. Optionally specify:
   - **Save Path**: Where to save the updated plan (e.g., `docs/specs/phase1.md`)
   - **System Prompt**: Top-level instructions for Claude when processing the plan
4. Choose your action:
   - **Approve** → Claude proceeds with implementation
   - **Request Changes** → Your annotations (and optionally the updated plan file) are sent back to Claude

### Save to File (New)

When you provide a save path:
- The plan is saved to the specified file relative to your repository root
- Claude receives your annotations as feedback but does NOT automatically start implementation
- You maintain control over when Claude implements the spec

### System Prompt (New)

Add a system prompt to provide high-level guidance to Claude when processing your annotated plan:
- Acts as a "system instruction" alongside your specific annotations
- Useful for setting context like "Focus on performance" or "Prioritize code maintainability"