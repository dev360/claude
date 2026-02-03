# Triage Toolkit

Claude Code plugin for issue investigation and self-improvement workflows.

## Installation

```bash
# Install globally (available in all projects)
claude plugin install /path/to/this/repo --scope user

# Or install for current project only
claude plugin install /path/to/this/repo --scope project
```

## Agents

### `triage`

Automated investigation for bug reports, support tickets, and incidents.

**Workflow:**
1. Parse ticket identifiers (env ID, timestamps, user info)
2. **STOP** - verify correct account with human
3. Gather evidence from logs, audit trails, session replays
4. Trace symptoms to code
5. **STOP** - present findings for validation
6. Document root cause with evidence

**Usage:**
```
> Use the triage agent to investigate this issue: [link or description]
```

### `retrospective`

Run ONLY after painful tasks to improve documentation and prompts.

**Triggers:**
- Multiple corrections needed
- Wasted effort from wrong assumptions
- Same problem twice

**Do NOT run** after smooth tasks.

**Usage:**
```
> Run a retrospective on what just happened
```

## Required MCP Servers

This plugin includes MCP server configs in `.mcp.json`. The agents need:

- **Browser automation** (`chrome-devtools` or `playwright`) - for navigating internal tools
- **Issue tracking** (`linear`, `github`, or similar) - for reading tickets

## Pre-approving Permissions

To avoid permission prompts for MCP tools, add to your settings:

**User-level** (`~/.claude/settings.json`):
```json
{
  "permissions": {
    "allow": [
      "mcp__chrome-devtools__*",
      "mcp__playwright__*",
      "mcp__linear__*"
    ]
  }
}
```

## Plugin Structure

```
.claude-plugin/
  plugin.json       # Plugin manifest
agents/
  triage.md         # Issue investigation agent
  retrospective.md  # Self-evolution agent
.mcp.json           # MCP server configurations
```
