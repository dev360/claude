# Triage Toolkit

## Agent Routing

| Task Type | Agent |
|-----------|-------|
| Bug investigation, support tickets, incidents | `triage` |
| After painful task (self-evolve) | `retrospective` |
| Code review before merge | `/review` skill → 9 parallel agents in `review/agents/` (scope-dependent) |

## Triage Agent

Use when investigating any "why did X happen?" question with a specific account.

**Human checkpoints** (agent will STOP at these):
1. Account verification - "Is this the right environment?"
2. Before sensitive access - "I'm about to view session replay"
3. Before direct access - "I need you to log in"
4. After analysis - "Does this match your understanding?"

## Retrospective Agent

Run ONLY after friction occurred. Do NOT run after smooth tasks.

**Triggers:**
- Multiple corrections needed
- Wrong assumptions wasted effort
- Same problem occurred twice
