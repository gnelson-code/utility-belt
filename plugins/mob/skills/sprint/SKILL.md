---
name: sprint
description: |
  End-to-end feature delivery: spec → test → plan → implement → PR.
  Orchestrates /mob:spec, /mob:test, plan generation, and /mob:swarm into a single autonomous pipeline.
  Default is fully autonomous. Use --checkpoints to pause between stages.
user-invocable: true
argument-hint: <feature-name> [--checkpoints] [--from-stage <spec|test|plan|implement|pr>]
---

You are the orchestrator of a sprint session. Your job is to take a feature from natural language description to a draft PR, orchestrating the full pipeline: specification, test generation, implementation planning, adversarial implementation, and PR creation.

You do not re-implement sub-skill logic. You delegate to the existing skills (`/mob:spec`, `/mob:test`, `/mob:swarm`) by following their defined processes and reading their state files. When a sub-skill's process changes, sprint inherits the change automatically.

---

## Setup

### Parse Arguments

- First argument: `<feature-name>` (required). Used to derive all output paths.
- `--checkpoints`: pause after spec, test, and plan stages for user review. Without this flag, the pipeline runs autonomously (except for two mandatory human touchpoints: the spec interview and plan approval).
- `--from-stage <stage>`: resume from a specific stage. Valid values: `spec`, `test`, `plan`, `implement`, `pr`.

### Derived Paths

- Spec: `specs/spec-<feature-name>.md`
- Plan: `plans/plan-<feature-name>.md`
- Sprint state: `.sprint-state.json`
- Sub-skill state files: `.spec-state.json`, `.test-state.json`, `.swarm-state.json`

### Check for Existing State

Look for `.sprint-state.json` in the project root. If it exists:
- If it references a different feature, warn the user before proceeding
- If all stages are complete, inform the user and ask if they want to re-run
- If `--from-stage` was specified, validate that required inputs exist on disk (see Input Validation below). Override the state file's current stage.
- If no `--from-stage`, resume from the first incomplete stage

If no state file exists, create one now with `current_stage: "spec"`.

### Input Validation for `--from-stage`

| Stage | Required Inputs |
|-------|----------------|
| `spec` | None |
| `test` | `specs/spec-<feature-name>.md` must exist with `Status: final` |
| `plan` | Spec file must exist. Test files from `.test-state.json` must exist. |
| `implement` | Spec, tests, and `plans/plan-<feature-name>.md` must exist. |
| `pr` | Implementation must be complete (`.swarm-state.json` shows all phases complete). |

If required inputs are missing, stop and report which inputs are missing and how to produce them.

---

## Stage 1: Spec

Follow the process defined in `skills/spec/SKILL.md`:

1. Run the interview phase — this is inherently interactive. The user answers questions to define the feature.
2. Spawn the `spec-writer` agent to draft the formal spec.
3. Run the `spec-gate` critique loop (maximum 5 iterations).
4. Finalize the spec to `specs/spec-<feature-name>.md`.

Track progress by reading `.spec-state.json`. When the spec stage completes (`.spec-state.json` status is `"complete"`), update `.sprint-state.json`:

```json
{
  "current_stage": "test",
  "stages": {
    "spec": {
      "status": "complete",
      "completed_at": "<ISO timestamp>",
      "output": "specs/spec-<feature-name>.md"
    }
  }
}
```

**Checkpoint:** If `--checkpoints` is active, pause after spec finalization:

```
## Sprint Checkpoint — Spec Complete

The spec has been finalized: specs/spec-<feature-name>.md

Review the spec before proceeding. Say "continue" to proceed to test generation,
or provide feedback to revise.
```

**Escalation:** If the spec stage escalates (loop 5 with unresolved issues), present the escalation and wait. The user can fix the issue and resume with `/sprint <feature-name> --from-stage spec` or provide guidance inline.

---

## Stage 2: Test

Follow the process defined in `skills/test/SKILL.md`:

1. Spawn the `test-writer` agent with the finalized spec path.
2. Run the `red-gate` critique loop (maximum 3 iterations, fully automated).
3. Commit the test files.

Track progress by reading `.test-state.json`. When the test stage completes, update `.sprint-state.json`:

```json
{
  "current_stage": "plan",
  "stages": {
    "test": {
      "status": "complete",
      "completed_at": "<ISO timestamp>",
      "test_files": ["<list from .test-state.json>"]
    }
  }
}
```

**Checkpoint:** If `--checkpoints` is active, pause after tests are committed:

```
## Sprint Checkpoint — Tests Complete

Failing test suite committed. Test files:
[list each file]

Review the tests before proceeding. Say "continue" to proceed to plan generation,
or provide feedback.
```

**Escalation:** If the test stage escalates (loop 3 with unresolved issues), present the escalation and wait.

---

## Stage 3: Plan

This stage generates the implementation plan. No existing skill handles this — sprint orchestrates it directly.

### Generate

1. Create the `plans/` directory if it doesn't exist.
2. Spawn the `plan-writer` agent as a fresh sub-agent.

**`plan-writer` prompt:**
```
Feature: [FEATURE NAME]

Spec path: specs/spec-<feature-name>.md

Test files:
[list each test file path — read from .sprint-state.json stages.test.test_files if available, otherwise fall back to .test-state.json test_files]

Read the spec and the test files. Explore the codebase to understand module structure,
dependencies, and conventions. Generate a phased implementation plan following the
template in your instructions.
```

3. Save the output to `plans/plan-<feature-name>.md`.

### User Approval

**The plan always requires user approval**, regardless of `--checkpoints` mode. A bad plan wastes the most compute of any artifact — it drives N phases of implementation with up to 3 critique loops each.

**Why there is no plan-gate:** The spec, test, and implementation stages all have automated critic loops. The plan does not. This is deliberate. A plan encodes judgment about decomposition, ordering, and scope — decisions that depend on context no critic agent has access to (team priorities, deployment constraints, what the user considers an acceptable first cut vs. a complete implementation). Automated critique can verify that a plan is internally consistent, but it cannot verify that it is *right*. The user is the only reviewer qualified to accept a plan, and requiring them to do so ensures they take accountability for what gets built.

Present the plan:

```
## Sprint — Plan Generated

The implementation plan has been generated: plans/plan-<feature-name>.md

[print the full plan contents inline]

Review the plan. You may:
- Say "continue" to proceed to implementation
- Provide feedback to revise the plan
- Say "regenerate" to discard and regenerate from scratch

The plan determines how implementation is structured. Review it carefully.
```

**If the user provides feedback:** Respawn the `plan-writer` with the original prompt plus the user's feedback appended. Increment `revisions` in the state file. Save the new output, present again.

**If the user says "regenerate":** Respawn the `plan-writer` with the original prompt (no prior context). Increment `revisions`. Save, present again.

**If the user says "continue":** Proceed to Stage 4.

Update `.sprint-state.json`:

```json
{
  "current_stage": "implement",
  "stages": {
    "plan": {
      "status": "complete",
      "completed_at": "<ISO timestamp>",
      "output": "plans/plan-<feature-name>.md",
      "revisions": 0
    }
  }
}
```

---

## Stage 4: Implement

Follow the process defined in `skills/swarm/SKILL.md` (the `/mob:swarm` command):

1. Pass `plans/plan-<feature-name>.md` as the plan file.
2. Let `/mob:swarm` run through all phases with its `architecture-gate` + `quality-gate` critique loops (maximum 3 loops per phase).
3. `/mob:swarm` manages its own `.swarm-state.json` and commits per phase.

Track progress by reading `.swarm-state.json`. When all phases complete, update `.sprint-state.json`:

```json
{
  "current_stage": "pr",
  "stages": {
    "implement": {
      "status": "complete",
      "completed_at": "<ISO timestamp>",
      "phases_completed": "<N>",
      "total_phases": "<N>",
      "total_loops": "<M>"
    }
  }
}
```

**No checkpoint** for implementation. Even in `--checkpoints` mode, implementation runs uninterrupted. Swarm has its own escalation mechanism if a phase fails at loop 3.

**Escalation:** If swarm escalates, present the escalation and wait. The user can fix the issue and resume with `/sprint <feature-name> --from-stage implement`.

---

## Stage 5: PR

After implementation completes:

1. Detect the current branch name.
2. Push to remote:
```bash
git push -u origin <branch>
```

3. Gather PR content from the state files:
   - Feature name and spec overview (read the spec's Overview section)
   - Test coverage from `.test-state.json` (`spec_item_counts` and `coverage`)
   - Phases completed from `.swarm-state.json`
   - Aggregated deferred items from all state files (`.spec-state.json`, `.test-state.json`, `.swarm-state.json`)

4. Create the draft PR:
```bash
gh pr create --draft --title "feat: <feature-name>" --body "$(cat <<'EOF'
## Summary

<One-paragraph feature description from the spec's Overview section>

## Artifacts

- **Spec:** specs/spec-<feature-name>.md
- **Plan:** plans/plan-<feature-name>.md
- **Tests:** <list of test file paths>

## Test Coverage

- TP: X/Y covered
- ERR: X/Y covered
- INV: X/Y covered
- PRE: X/Y covered
- EDGE: X/Y covered

## Implementation

<N> phases completed in <M> total loops.

[If deferred items exist from any stage:]
## Deferred Items

### From Spec
- [Loop N, section] <description>

### From Tests
- [Loop N] <description>

### From Implementation
- [Phase N, Loop N, source] <description>

---

Generated by /sprint
EOF
)"
```

5. Update `.sprint-state.json`:

```json
{
  "current_stage": "complete",
  "stages": {
    "pr": {
      "status": "complete",
      "completed_at": "<ISO timestamp>",
      "url": "<PR URL>",
      "branch": "<branch name>"
    }
  }
}
```

6. Present the final summary:

```
## Sprint Complete

Feature: <feature-name>
PR: <URL>

### Pipeline Summary
- Spec: X loops, finalized
- Tests: X loops, Y test files
- Plan: Z phases defined [N revisions]
- Implementation: N phases, M total loops
- PR: draft created

[If deferred items exist:]
### Deferred Items
[aggregated from all state files — spec, test, swarm]

The draft PR is ready for review.
```

---

## State File

Maintain `.sprint-state.json` in the project root throughout the run. Write it after every stage transition — this is what allows recovery after context compaction or interruption.

**Transition pattern:** When entering a stage, immediately set that stage's `status` to `"in_progress"` and `started_at` to the current timestamp. When the stage completes, set `status` to `"complete"` and `completed_at`. This ensures that if the process is interrupted mid-stage, the state file reflects which stage was running, not just which stages are done. The JSON examples in each stage section above show only the completed state for brevity — the `"in_progress"` write happens first.

```json
{
  "feature": "<feature-name>",
  "started": "<ISO timestamp>",
  "last_updated": "<ISO timestamp>",
  "current_stage": "spec",
  "checkpoints": false,
  "stages": {
    "spec": {
      "status": "pending",
      "started_at": null,
      "completed_at": null,
      "output": null,
      "escalated": false
    },
    "test": {
      "status": "pending",
      "started_at": null,
      "completed_at": null,
      "test_files": [],
      "escalated": false
    },
    "plan": {
      "status": "pending",
      "started_at": null,
      "completed_at": null,
      "output": null,
      "revisions": 0
    },
    "implement": {
      "status": "pending",
      "started_at": null,
      "completed_at": null,
      "phases_completed": 0,
      "total_phases": 0,
      "total_loops": 0,
      "escalated": false
    },
    "pr": {
      "status": "pending",
      "completed_at": null,
      "url": null,
      "branch": null
    }
  }
}
```

Valid `status` values per stage: `"pending"`, `"in_progress"`, `"complete"`, `"escalated"`.

Valid `current_stage` values: `"spec"`, `"test"`, `"plan"`, `"implement"`, `"pr"`, `"complete"`.

---

## Notes

- The spec interview and the plan approval are the two mandatory human touchpoints. Everything else runs autonomously unless `--checkpoints` is active.
- Sprint does not manage git branches. The user must be on the correct branch before invoking. This matches `/mob:swarm`'s existing contract.
- Sprint does not re-implement sub-skill logic. It follows their processes and reads their state files. If a sub-skill evolves, sprint inherits the change.
- Always write to `.sprint-state.json` after every stage transition — do not rely on conversational context alone.
- `--from-stage` validates that required inputs exist before proceeding. Missing inputs are reported, not guessed.
- If any sub-skill escalates, sprint presents the escalation and waits. The user can fix and resume with `--from-stage`.
- The PR is always created as a draft. The user promotes it when ready.
