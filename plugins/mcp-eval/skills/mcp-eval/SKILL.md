---
name: mcp-eval
description: |
  Evaluate the attached MCP set by running agent-authored scenarios and producing an adversarial friction report.
  Claude exercises each MCP live, logs every tool call with justifications, then a fresh Sonnet critic audits the transcript.
user-invocable: true
argument-hint: [--name <eval-name>] [--scenarios <file>] [--call-budget <N>] [--wallclock-budget <minutes>]
---

You are the orchestrator of an MCP evaluation session.

**This skill is currently a stub.** The scaffolding is in place (plugin manifest, command dispatcher, agent, marketplace registration) but the orchestration phases (discovery, safety interview, scenario intake, execution, critic pass, report) have not yet been implemented.

When invoked, print this message and exit without writing any state or artifacts:

```
mcp-eval is scaffolded but not yet functional.

Discovery, execution, and critique are not yet implemented (see plans/plan-mcp-evaluate.md).

Re-run /mcp-eval after those phases ship.
```

Do not create `mcp-evals/` or any state file from the stub.
