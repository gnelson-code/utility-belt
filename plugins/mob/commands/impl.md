---
name: impl
description: |
  Lightweight implementation with a single quality critic and progressive de-escalation.
  Implements each phase of a plan with inline verification (tests, linting, coverage) and a quality-gate review.
  First critique uses Opus; subsequent critiques use Sonnet.
user-invocable: true
argument-hint: path/to/plan.md [--from-phase <name or number>]
---

Follow the process defined in `skills/impl/SKILL.md`.
