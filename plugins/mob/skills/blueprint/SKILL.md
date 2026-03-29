---
name: blueprint
description: |
  Collaborative plan creation with adversarial hardening.
  Draft an implementation plan in plan mode, then harden it with a Sonnet critic before writing the final output.
user-invocable: true
argument-hint: <feature-name>
---

You are the orchestrator of a blueprint session. Your job is to collaborate with the user on an implementation plan, then harden it with an adversarial critic before finalizing.

This is the lightweight alternative to the full spec → test → plan pipeline. No formal spec. No test-first. Just a solid plan, stress-tested by a critic, ready for `/mob:impl`.

---

## Setup

### Parse Arguments

- First argument: `<feature-name>` (required). Used to derive the output path: `plans/plan-<feature-name>.md`

If the `plans/` directory does not exist, create it when writing the final plan.

---

## Phase 1: Plan Mode

Enter plan mode. Your goal is to produce a draft implementation plan collaboratively with the user.

### Explore the Codebase

Before drafting anything, build context:

1. **CLAUDE.md or equivalent**: Read project-level instructions first.
2. **Module structure**: Directory layout, package boundaries, entry points. Understand where things live and why.
3. **Similar features**: Find the closest existing feature to what's being built. Study its file organization, dependency patterns, and integration.
4. **Dependencies**: Libraries, frameworks, and internal modules the feature will interact with. Read their interfaces.
5. **Build and test toolchain**: How is code compiled, tested, linted? The plan must produce code that passes the existing toolchain.
6. **Import patterns**: How do existing modules import each other? File placements must work naturally with existing imports.

### Draft the Plan

Draft a plan following this template. Present it to the user inline for collaboration.

```markdown
# Plan: <Feature Name>

**Created:** <ISO date>

## Phase 1: <Name>

<Goal paragraph — what this phase accomplishes, why it comes first, and what it enables for subsequent phases.>

### Tasks
- Create `<path/to/file>`: <what this file contains and why>
- Modify `<path/to/file>`: <what changes and why>

### Tests Targeted
- <Which tests should pass or get closer to passing after this phase>

### Acceptance
- <Concrete verification: what must be true when this phase is done>

## Phase 2: <Name>

...

## Deferred / Out of Scope

<Anything explicitly not addressed by this plan.>
```

**Rules:**
- Every task references a concrete file path — not "create a service" but "create `src/services/feature.ts`"
- Every phase has at least one testable outcome in Tests Targeted
- A phase that touches more than 8 files is too large — split it
- Tasks within a phase are ordered by dependency
- Each phase must be independently committable
- The Deferred section must exist even if empty

### Collaborate

Work with the user to refine the plan. They may:
- Ask to restructure phases
- Add or remove scope
- Question ordering decisions
- Request more detail on specific tasks

Iterate until the user approves the plan. Then exit plan mode.

---

## Phase 2: Adversarial Critique

Spawn the `plan-critic` agent as a fresh sub-agent. Pass it the full plan text.

**`plan-critic` prompt:**
```
Feature: [FEATURE NAME]

[paste the full plan document]

Review this implementation plan. Explore the codebase to verify file paths, conventions, and dependencies. Use the severity classification in your instructions.
```

Wait for the critic to complete.

### Triage

Parse the critic's output. For each issue:

| Severity | Action |
|----------|--------|
| Critical | Must fix. Investigate and correct the plan. |
| Major | Must fix unless your investigation shows the critic is wrong. If overriding, note why. |
| Minor | Fix if tractable. Otherwise note in Deferred section. |
| Suggestion | Ignore. Do not add scope. |

Apply fixes directly to the plan. Do not re-run the critic — this is a single pass.

If the critic found Critical or Major issues, briefly tell the user what was changed and why.

---

## Phase 3: Finalize

1. Create the `plans/` directory if it doesn't exist.
2. Write the final plan to `plans/plan-<feature-name>.md`.
3. Print the final plan to the terminal.
4. Print a summary:

```
## Blueprint Complete

Plan written to: plans/plan-<feature-name>.md

Phases: <N>
Critic findings: <N Critical, N Major fixed / N Minor deferred>

Ready for implementation:
  /mob:impl plans/plan-<feature-name>.md
```

---

## Notes

- The plan-critic runs once. No iteration loop. This is the lightweight path — if you need formal spec-level rigor, use `/mob:sprint`.
- Do not add scope during the critic triage. If the critic suggests new features, that's a Suggestion — ignore it.
- The plan must be consumable by `/mob:impl` — same phase structure, same task format.
