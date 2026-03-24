---
name: sprint
description: |
  End-to-end feature delivery: spec → test → plan → implement → PR.
  Orchestrates /mob:spec, /mob:test, plan generation, and /mob:swarm into a single autonomous pipeline.
  Default is fully autonomous. Use --checkpoints to pause between stages.
user-invocable: true
argument-hint: <feature-name> [--checkpoints] [--from-stage <spec|test|plan|implement|pr>]
---

Follow the process defined in `skills/sprint/SKILL.md`.
