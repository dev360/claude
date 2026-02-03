# Claude Plugins

Personal Claude Code agents for investigation and self-improvement workflows.

## Installation

```bash
# Add marketplace
claude plugin marketplace add dev360/claude

# Install plugin
claude plugin install plugins@dev360
```

## Agents

### `triage`

Automated investigation for bug reports, support tickets, and incidents.

**Workflow:**
1. Parse ticket identifiers
2. **STOP** - verify correct account with human
3. Gather evidence from logs, audit trails, session replays
4. Trace symptoms to code
5. **STOP** - present findings for validation
6. Document root cause with evidence

### `retrospective`

Run ONLY after painful tasks to improve documentation and prompts.

**Triggers:**
- Multiple corrections needed
- Wasted effort from wrong assumptions
- Same problem twice

**Do NOT run** after smooth tasks.

## Pre-approving Permissions

To avoid permission prompts for MCP tools, add to `~/.claude/settings.json`:

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
