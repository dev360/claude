# Claude Agents

Personal Claude Code agents for investigation and self-improvement workflows.

## Installation

### Option 1: User-level agents (recommended)

Copy agents to your Claude Code user directory:

```bash
mkdir -p ~/.claude/agents
cp .claude/agents/*.md ~/.claude/agents/
```

### Option 2: Project-level

Clone this repo and work from within it, or symlink `.claude/` to your projects.

## Agents

### `/triage` - Issue Investigation

Automated triage for bug reports, support tickets, and incidents.

**Workflow:**
1. Parse ticket identifiers
2. STOP - verify correct account with human
3. Gather evidence (logs, audit trails, session replays)
4. Trace to code
5. STOP - present findings for human validation
6. Document root cause

**Required MCP servers:**
- `chrome-devtools` or `playwright` - browser automation for internal tools
- `linear` (or github/jira) - issue tracking

### `/retrospective` - Self-Evolution

Run ONLY after painful tasks to improve documentation and prompts.

**Triggers:**
- Multiple corrections needed
- Wasted effort from wrong assumptions
- Same problem twice
- Knowledge gaps

**Do NOT run** after smooth tasks.

## MCP Configuration

The `.mcp.json` includes common servers. Copy to your home directory or project:

```bash
cp .mcp.json ~/.mcp.json
```

Pre-approved permissions are in `.claude/settings.json`.

## Usage

```bash
# In any project with these agents available
claude

# Then use the triage agent
> /triage https://linear.app/team/issue/123
> /triage "Investigate why newsletter stopped sending for account 12345"

# After a painful session
> /retrospective
```
