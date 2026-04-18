---
name: mcp-eval
description: |
  Evaluate the attached MCP set by running agent-authored scenarios and producing an adversarial friction report.
  Claude exercises each MCP live, logs every tool call with justifications, then a fresh Sonnet critic audits the transcript.
user-invocable: true
argument-hint: [--name <eval-name>] [--scenarios <file>] [--call-budget <N>] [--wallclock-budget <minutes>]
---

Follow the process defined in `skills/mcp-eval/SKILL.md`.
