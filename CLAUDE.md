# Claude Agents

Global agents for issue investigation and self-improvement workflows.

## Agent Routing

| Task Type | Agent |
|-----------|-------|
| Bug investigation, support tickets, incidents | `.claude/agents/triage.md` |
| After painful task (self-evolve) | `.claude/agents/retrospective.md` |

## Triage Agent

Use `/triage` or invoke the triage agent when:
- Investigating a bug report from issue tracker (Linear, Jira, GitHub Issues)
- Support ticket escalation
- Production incident investigation
- Any "why did X happen?" question with a specific account/environment

The agent will:
1. Parse ticket identifiers (env ID, timestamps, user info)
2. **STOP for verification** - confirm correct account
3. Gather evidence from logs, audit trails, session replays
4. Trace to code
5. **STOP for verification** - present findings
6. Document root cause with evidence

### Required Tools

The triage agent needs browser automation to access internal tools:
- `chrome-devtools` MCP - for navigating observability/admin tools
- `linear` MCP - for issue tracking (or equivalent)

### Human Checkpoints

The agent WILL STOP and ask for verification at:
1. Account identification - "Is this the right environment?"
2. Before sensitive access - "I'm about to view session replay"
3. Before direct access - "I need you to log into the admin tool"
4. After analysis - "Does this explanation match your understanding?"

## Retrospective Agent

Use `/retrospective` or invoke ONLY after encountering friction:
- Had to correct Claude multiple times
- Wasted effort due to wrong assumptions
- Same problem occurred twice
- Knowledge gap caused mistakes

The agent will:
1. Identify what went wrong
2. Find the smallest change to prevent recurrence
3. Update the relevant documentation/agent prompts

**Do NOT run after smooth tasks.**

## MCP Server Setup

This repo includes `.mcp.json` with common MCP servers. The tools are pre-approved in `.claude/settings.json`.

To use these agents globally, symlink or copy to your home directory:
```bash
# Option 1: Symlink (recommended)
ln -sf ~/Projects/claude/.claude ~/.claude-global
ln -sf ~/Projects/claude/.mcp.json ~/.mcp.json

# Option 2: Copy to Claude's user config
cp -r ~/Projects/claude/.claude/agents ~/.claude/agents
```
