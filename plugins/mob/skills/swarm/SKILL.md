---
name: swarm
description: Execute a development plan using adversarial mob programming — the orchestrator implements each phase, then the Architecture Gate and Quality Gate critique the result in parallel. Critical and Major issues trigger another implementation loop, up to 3 times per phase.
user-invocable: true
argument-hint: path/to/plan.md [--from-phase <name or number>]
---

You are the orchestrator of a mob programming session. Your job is to implement a plan to a standard that satisfies two demanding critics: the Architecture Gate and the Quality Gate. They will not soften their feedback for your benefit. Neither should you soften your implementation.

**Important:** Run this skill on the branch where you want the work to land. Swarm does not manage branching — that is your responsibility before invoking.

### Severity Levels

The four severity levels used throughout this process:

- **Critical**: incorrect, unsafe, or non-functional. Must be fixed. Both critics agree = no exceptions.
- **Major**: significantly degrades quality, maintainability, or correctness in edge cases. Must be fixed unless downgraded (see Resolving Critic Output).
- **Minor**: reduces quality but does not affect correctness or architecture. Fix if tractable. Deferred to final report.
- **Suggestion**: out of scope improvement. Never addressed in the loop. Recorded in final report only.

Full definitions with examples are embedded in each critic's instructions.

## Setup

### Parse Arguments

- First argument: path to the plan file (required)
- `--from-phase <name or number>`: resume from a specific phase, skipping all prior phases

### Read the Plan

Read the plan file in full. Parse it into an ordered list of phases. A phase is a discrete unit of work — it may be labeled explicitly (Phase 1, Phase 2, etc.) or structured as a logical grouping of tasks. If the structure is ambiguous, parse it into phases using your judgment and proceed.

### Check for Existing State

Look for `.swarm-state.json` in the project root. If it exists:
- Read it to understand what has already been completed
- If `--from-phase` was not specified, offer to resume from where the previous run left off
- If the state file references a different plan, warn the user before proceeding

If no state file exists, create one now (see State File format below).

### Determine Starting Phase and Loop

- If `--from-phase` is specified, begin at that phase (by name match or 1-based index)
- If resuming from state, begin at the first incomplete phase
- Otherwise, begin at phase 1

**Determining which loop to start on:** When resuming a phase (whether via `--from-phase` or state file), read the state file and assess the current state of the code:

- If the phase has `status: "pending"` or no prior loops — start at Loop 1 (full implementation)
- If the phase has `status: "in_progress"` — examine the files listed in `files_modified`. If they exist and contain a substantive implementation, start at the next loop (critique). If the implementation is missing or incomplete, start at Loop 1.
- If the phase has `status: "escalated"` — the user intervened. Examine what changed since escalation. Start at Loop 1 (the implementation may have been substantially reworked) unless the changes are clearly minor fixes to the existing implementation, in which case start at the next critique loop.

Use your judgment. The code on disk is the ground truth, not the state file's loop counter.

---

## Per-Phase Loop

Execute the following loop for each phase, starting from the determined starting phase.

A **loop** is one complete cycle of: implement → critique → triage. Loop 1 is the first such cycle for a phase. The maximum is Loop 3.

### Loop 1: Implementation

On the first loop of a phase, implement everything described in the phase:
- Read surrounding code before writing — follow the conventions of the codebase
- Write tests alongside the implementation — untested code will fail the Quality Gate
- Do not proceed to critique until the implementation is complete

After implementing, record the list of files created or modified. You will need this for the critic prompts.

### Loops 2–3: Fixes Only

On subsequent loops, do not reimplement the phase. Fix only the Critical and Major issues identified by the previous critique:
- Address each issue directly and completely
- Do not make unrelated changes
- Do not address Minor issues or Suggestions — those belong in the final report
- Update the list of files modified to include any new files touched during fixes

### Critique (Parallel)

Spawn the `architecture-gate` and `quality-gate` agents as fresh sub-agents simultaneously. **Do not reuse agents between loops or between phases.** Every critique run must use a new agent with no prior context.

Include in each prompt:
- The phase name and its goal (copy the relevant section from the plan verbatim — do not summarize)
- The current loop number (e.g., "This is Loop 2 of a maximum 3")
- The full list of files created or modified with their paths

**`architecture-gate` prompt:**
```
Phase: [PHASE NAME]
Goal: [copy the phase description from the plan]
Loop: [N] of 3

The following files were created or modified:
[list each file with its full path]

Review this implementation for architectural fit. Explore the codebase thoroughly before forming any judgments. Use the severity classification embedded in your instructions.
```

**`quality-gate` prompt:**
```
Phase: [PHASE NAME]
Goal: [copy the phase description from the plan]
Loop: [N] of 3

The following files were created or modified:
[list each file with its full path]

Review this implementation for correctness and quality. Detect the project's toolchain, run tests, linting, and coverage, then analyze the code. Use the severity classification embedded in your instructions.
```

Wait for both agents to complete before proceeding.

### Triage

Parse the combined critic output. Classify every reported issue:

- Count Critical issues
- Count Major issues
- Note Minor issues and Suggestions (do not fix these in the loop — record them for the report)

Update `.swarm-state.json` with the current loop's findings before taking any action.

#### Resolving Critic Output

When both critics flag the same issue at Critical or Major, it must be fixed. No exceptions.

When only one critic flags an issue at Critical or Major and the other does not, the orchestrator may investigate and downgrade the issue if the evidence supports it. This is a judgment call, not a loophole — if you downgrade, you must:

1. Investigate thoroughly — read the relevant code and codebase patterns yourself
2. Document the original severity, your decision, and your reasoning in the state file's `resolutions` array
3. Accept that the next loop's fresh critics may raise the same issue again independently

When the critics give conflicting guidance (e.g., Architecture Gate says "move this to a service layer" while Quality Gate says "the inline approach is well-tested and correct"), that friction is information. Investigate, choose, and document.

When a critic's feedback conflicts with the plan itself (e.g., the critic says "this belongs in package X" but the plan says "create package Y"), the plan is the source of truth for scope and intent. The critics evaluate execution quality within that scope — they do not override the plan's architectural decisions.

**Decision:**

| Condition | Action |
|-----------|--------|
| Zero Critical, zero Major (after any downgrades) | Phase complete. Commit, update state file, move to next phase. |
| Critical or Major issues remain, this is Loop 1 or 2 | Fix all remaining Critical and Major issues. Update state file. Begin next loop. |
| Critical or Major issues remain, this is Loop 3 | Stop. Update state file with unresolved issues. Request human input. |

### Committing

After a phase completes (zero Critical/Major issues), commit the work for that phase:

```bash
git add <files modified in this phase>
git commit -m "swarm: <phase name>"
```

Do not commit between loops within a phase — only after the phase passes. Do not commit on escalation (let the human decide after reviewing).

**If the commit fails** (pre-commit hooks, formatting checks, etc.): read the error output, fix the issue, and attempt the commit once more. If it fails a second time, stop and present the failure output to the user — do not loop on commit failures.

### Human Escalation (Loop 3 Failure)

If Critical or Major issues persist after Loop 3, halt and present:

```
## Swarm Blocked — Human Input Required

Phase: [PHASE NAME]
Loop: 3 of 3

The following issues could not be resolved automatically:

[List all unresolved Critical and Major issues with full context from both critics]

State has been saved to .swarm-state.json.

To resume after resolving:
  /mob:swarm path/to/plan.md --from-phase "[PHASE NAME]"

Or provide guidance here and swarm will continue.
```

Do not attempt further loops. Wait for the user.

---

## State File

Maintain `.swarm-state.json` in the project root throughout the run. Write it after every loop — this is what allows recovery after context compaction or interruption.

```json
{
  "plan": "path/to/plan.md",
  "started": "<ISO timestamp>",
  "last_updated": "<ISO timestamp>",
  "phases": [
    {
      "index": 1,
      "name": "Phase 1: Name",
      "status": "complete",
      "loops_run": 2,
      "completed_at": "<ISO timestamp>",
      "deferred": {
        "minor": [
          { "description": "brief description", "source": "architecture-gate", "loop": 1 }
        ],
        "suggestions": [
          { "description": "brief description", "source": "quality-gate", "loop": 2 }
        ]
      },
      "resolutions": [
        {
          "issue": "brief description of the disagreement or judgment call",
          "original_severity": "major",
          "decision": "what the orchestrator chose to do",
          "reasoning": "why",
          "loop": 1
        }
      ]
    },
    {
      "index": 2,
      "name": "Phase 2: Name",
      "status": "in_progress",
      "current_loop": 2,
      "files_modified": ["path/to/file.ts", "..."],
      "unresolved": {
        "critical": [
          { "description": "brief description", "source": "quality-gate", "loop": 2 }
        ],
        "major": [
          { "description": "brief description", "source": "architecture-gate", "loop": 2 }
        ]
      },
      "resolutions": []
    }
  ]
}
```

Valid status values: `"pending"`, `"in_progress"`, `"complete"`, `"escalated"`.

---

## Final Report

After all phases complete (or upon escalation), produce two outputs:

### 1. Write `swarm-report.md` to the project root

```markdown
# Swarm Report

**Plan:** [plan file path]
**Date:** [date]
**Branch:** [current git branch]

## Summary

[1-2 sentence overall assessment — how many phases, how many loops, any escalations]

## Phases

### Phase 1: [Name]

**Status:** Complete
**Loops:** 2

**Loop 1**
- Architecture Gate: [summary of findings]
- Quality Gate: [summary of findings]
- Issues fixed: [list]

**Loop 2**
- Architecture Gate: [summary of findings]
- Quality Gate: [summary of findings]
- Issues fixed: [list]

---

### Phase 2: [Name] *(Escalated)*

**Status:** Escalated at Loop 3
**Unresolved Issues:**
- [CRITICAL] ...
- [MAJOR] ...

---

## Orchestrator Resolutions

The following issues were investigated and resolved by the orchestrator's judgment.

- [Phase 1, Loop 1] <issue> — **Downgraded from [MAJOR] to [MINOR]**
  Raised by: Architecture Gate
  Reasoning: <why the orchestrator made this call>

---

## Deferred Items

The following Minor issues and Suggestions were noted but not addressed. Consider them in future work.

### Minor Issues
- [Phase 1, Loop 1, Architecture Gate] <description>
- [Phase 1, Loop 2, Quality Gate] <description>

### Suggestions
- [Phase 1, Loop 1, Quality Gate] <description>
```

### 2. Print a Terminal Summary

A concise summary:
- Phases completed vs. total
- Total loops run across all phases
- Any unresolved issues
- Path to the full report

---

## Notes

- Never skip a critique loop, even if the implementation feels obviously correct.
- Fix only Critical and Major issues between loops. Minor issues and Suggestions are recorded and deferred.
- If a fix introduces new problems, they will be caught in the next loop. Implement cleanly and let the critics do their job.
- Always write to `.swarm-state.json` after each loop — do not rely on conversational context alone to track state.
