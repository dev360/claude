# dev360 Claude Plugins

## Install

```bash
# Add marketplace
claude plugin marketplace add dev360/claude

# Install plugins
claude plugin install dev@dev360 --scope user
claude plugin install review@dev360 --scope user
claude plugin install debugger@dev360 --scope user
```

## Plugins

### `dev`

Investigation and self-improvement agents.

- **triage** - Automated bug/incident investigation. Parses tickets, gathers evidence from logs and session replays, traces symptoms to code. Stops for human verification at key checkpoints.
- **retrospective** - Post-mortem agent that runs after painful tasks to improve docs and prompts. Only triggers when corrections were needed or effort was wasted.

### `review`

Code review via 9 parallel specialized agents. Invoked with `/review`.

Agents are selected based on change size:
- **Small changes** (1-50 lines): `logic`, `boundary`, `error-handling`, `security`
- **Medium changes** (51-300 lines): adds `data-flow`, `contracts`, `test-gaps`
- **Large changes** (300+ lines): adds `idioms`, `architecture`

Each agent hunts for specific issues — logic errors, edge cases, missing error handling, security holes, blast radius, breaking API changes, test gaps, non-idiomatic patterns, and code smells. Findings are consolidated with pass/warn/fail verdicts per agent.

### `debugger`

Browser debugging via Chrome DevTools MCP. Launches a subagent that can navigate pages, click elements, fill forms, monitor network requests, capture console errors, and take screenshots. Used automatically when Claude needs eyes in the browser, or explicitly via `/debug`.
