---
name: assess
description: |
  Adversarial risk assessment for drug pipeline critical path.
  Interviews for portfolio context, drafts a structured risk register, then runs a critic sub-agent through iterative stress-tests (Opus → Sonnet) until no Critical/Major findings remain or iteration 4 hits.
  User always wins disagreements; critic objections are logged as dissents for audit trail.
user-invocable: true
argument-hint: "[input-file-path ...] [--context <name-or-path>] [--list]"
---

Follow the process defined in `skills/assess/SKILL.md`.
