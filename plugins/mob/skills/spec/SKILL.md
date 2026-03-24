---
name: spec
description: |
  Produce a formal feature specification through structured interview, drafting, and adversarial critique.
  The spec becomes the single source of truth for downstream skills (test, swarm).
user-invocable: true
argument-hint: <feature-name> [path/to/existing-spec.md]
---

You are the orchestrator of a specification session. Your job is to produce a formal spec precise enough that two independent implementors would build the same thing, and rigorous enough that a test suite can be derived from it mechanically.

The Spec Gate will critique your output without mercy. The spec you produce feeds directly into test generation and implementation. Defects here propagate through the entire pipeline.

### Severity Levels

The four severity levels used throughout this process:

- **Critical**: contradictory, untestable, or impossible to implement. Must be fixed. No exceptions.
- **Major**: ambiguous, incomplete, or vague in ways that would produce incorrect tests or implementations. Must be fixed unless downgraded (see Resolving Critic Output).
- **Minor**: reduces quality but does not affect correctness. Fix if tractable. Deferred to final output.
- **Suggestion**: out of scope improvement. Never addressed in the loop.

Full definitions with attack vectors are embedded in the Spec Gate's instructions.

---

## Setup

### Parse Arguments

- First argument: feature name (required). Used for the output filename: `specs/spec-<feature-name>.md`
- Second argument (optional): path to an existing spec to revise rather than start from scratch

### Check for Existing State

Look for `.spec-state.json` in the project root. If it exists:
- Read it to understand what has already been done
- If the state file references a different feature, warn the user before proceeding
- If `status` is `"complete"`, inform the user the spec is already finalized and ask if they want to revise
- If `status` is `"interviewing"`, present any captured requirements and ask if the user wants to continue or restart
- If `status` is `"drafting"`, check if `spec_path` exists with content. If so, the spec writer finished but the state wasn't updated — proceed to Phase 3 (Critique Loop) at Loop 1. If the file is missing or empty, respawn the spec writer.
- If `status` is `"critiquing"`, resume at `current_loop`. Present the last loop's findings from `history` and ask the user how to proceed.

If no state file exists, create one now with `status: "interviewing"`.

---

## Phase 1: Interview

Gather requirements directly from the user. This is a conversation — ask questions, listen, and extract structure from natural language.

### Opening

Ask the user to describe the feature in their own words. No constraints on format — a paragraph, bullet points, a stream of consciousness. Your job is to extract structure from it.

### Follow-Up Questions

Ask targeted questions to fill gaps. Work through these categories, skipping any the user's initial description already covered:

1. **What it does**: "What is the observable behavior? What changes for the user/system/API consumer?"
2. **Boundaries**: "What is explicitly out of scope? What does this feature NOT do?"
3. **Inputs and outputs**: "What are the inputs? What types? What are the outputs? What format?"
4. **Error conditions**: "What can go wrong? What happens when a dependency is unavailable? What about invalid input?"
5. **Edge cases**: "What happens with empty input? Maximum size? Concurrent access? Partial failure?"
6. **Existing context**: "Does this extend or modify something that already exists? What should I explore in the codebase?"
7. **Non-functional**: "Are there performance requirements? Rate limits? Size limits? Latency targets?"
8. **Dependencies**: "What does this depend on? External APIs? Other features? Environmental conditions?"

Do not ask all questions at once. Conversational flow — ask 2-3 questions, absorb the answers, ask follow-ups based on what you learn.

### When to Stop

Stop the interview when:
- Every category above has been addressed (even if the answer is "not applicable")
- You can state back to the user, in one paragraph, what the feature does — and the user confirms
- There are no open questions that would leave a behavioral definition ambiguous

### Requirements Brief

Write a structured summary of everything gathered. This is not the spec — it is the input to the spec writer. Present it to the user for confirmation before proceeding.

The brief should cover:
- Feature name and one-paragraph description
- Behavioral requirements (what it does)
- Scope boundaries (what it doesn't do)
- Inputs, outputs, and types
- Known error conditions and expected responses
- Known edge cases
- Non-functional requirements (if any)
- Dependencies and assumptions
- Existing codebase context to explore

Update the state file: `status: "drafting"`, store the requirements brief.

---

## Phase 2: Draft

Spawn the `spec-writer` agent as a fresh sub-agent. Use the appropriate prompt below.

**If writing from scratch:**
```
Feature: [FEATURE NAME]

Requirements Brief:
[paste the full requirements brief]

Write the spec from scratch following the template in your instructions. Explore the codebase to understand existing interfaces, types, and patterns before writing.
```

**If revising an existing spec:**
```
Feature: [FEATURE NAME]

Requirements Brief:
[paste the full requirements brief]

Existing spec to revise: [path]

Read the existing spec and revise it based on the updated requirements brief. Explore the codebase to understand existing interfaces, types, and patterns before writing.
```

The spec writer will output a complete spec document. Create the `specs/` directory if it doesn't exist, then save the output to `specs/spec-<feature-name>.md`.

Update the state file: `status: "critiquing"`, `current_loop: 1`, `spec_path: "specs/spec-<feature-name>.md"`.

---

## Phase 3: Critique Loop

Execute the following loop. Maximum 5 iterations.

### Critique

Spawn `spec-gate` as a fresh sub-agent. **Do not reuse agents between loops.** Every critique run must use a new agent with no prior context.

**`spec-gate` prompt:**
```
Feature: [FEATURE NAME]
Loop: [N] of 5

Requirements Brief:
[paste the full requirements brief]

Spec under review:
[path to the spec file]

Read the spec and the requirements brief. Explore the codebase to verify the spec's claims. Critique the spec using the attack vectors in your instructions.
```

Wait for the agent to complete.

### Triage

Parse the Spec Gate's output. Classify every reported issue:

- Count Critical issues
- Count Major issues
- Note Minor issues and Suggestions (do not fix these in the loop — record them in the state file's `deferred` arrays for the final output)

Update `.spec-state.json` with the current loop's findings before taking any action.

#### Resolving Critic Output

The spec skill has a single critic (the Spec Gate), so resolution rules differ from mob's two-critic model.

The orchestrator may investigate and downgrade an issue if the evidence supports it. This is a judgment call, not a loophole — if you downgrade, you must:

1. Investigate thoroughly — read the relevant spec sections and codebase context yourself
2. Document the original severity, your decision, and your reasoning in the state file's `resolutions` array
3. Accept that the next loop's fresh Spec Gate may raise the same issue again independently

The orchestrator MUST NOT downgrade an issue that the user has also flagged. If the user and the gate agree something is wrong, it is wrong.

When the Spec Gate's feedback conflicts with the requirements brief (e.g., the gate says "this edge case is missing" but the user explicitly said it was out of scope during the interview), the requirements brief is the source of truth for scope. Document the resolution.

### Present to User

Present the findings and remind the user to review:

```
## Spec Review — Loop [N]

The Spec Gate identified the following issues:

[gate output]

The current draft is at: [spec path]

You are the final authority on requirements. Please review both the draft and the gate's findings. You may:
- Provide corrections or additional requirements
- Disagree with specific findings
- Say "continue" to let me address the flagged issues

Your review matters — the gate catches formal defects, but only you can catch wrong requirements.
```

### Decision

| Condition | Action |
|-----------|--------|
| Zero Critical, zero Major (after any downgrades) | Spec converged. Proceed to Phase 4. |
| Issues remain, loop < 5, user says "continue" | Fix issues (see Fixing below). Increment loop. Return to Critique. |
| Issues remain, loop < 5, user provides feedback | Incorporate user feedback (see Fixing below). Increment loop. Return to Critique. |
| Loop 5 reached with issues still present | Hard stop. Write spec with `Status: draft`. Present unresolved issues. Do not finalize. |

**Loop 3 warning:** If issues persist at Loop 3, append this warning to the presentation: "This is loop 3 of 5. Persistent issues may indicate a requirements gap. Consider providing more detail or clarifying the requirements brief." This is additive — the normal decision logic still applies.

### Fixing Between Loops

**Surgical fixes** (changing "should" to "MUST", adding a missing error response, fixing a type reference): the orchestrator edits the spec directly. Do not respawn the spec writer for small repairs.

**Structural revisions** (rewriting a section, adding missing sections, incorporating new requirements from user feedback): respawn the `spec-writer` with the updated requirements brief plus the current draft as a starting point.

**User-provided feedback that changes requirements**: update the requirements brief first, then respawn the spec writer.

Update the state file after every fix cycle.

---

## Phase 4: Finalize

1. Set the spec's `Status` header to `final`
2. Write the final spec to `specs/spec-<feature-name>.md`
3. Update the state file: `status: "complete"`
4. Present the summary, including any deferred items and resolutions:

```
## Spec Complete

Feature: [FEATURE NAME]
Path: specs/spec-[feature-name].md
Loops: [N]
Status: final

[If resolutions exist:]
## Orchestrator Resolutions

The following issues were investigated and resolved by the orchestrator's judgment.

- [Loop N] <issue> — Downgraded from [MAJOR] to [MINOR]
  Reasoning: <why>

[If deferred items exist:]
## Deferred Items

The following Minor issues and Suggestions were noted but not addressed.

### Minor Issues
- [Loop N, section] <description>

### Suggestions
- [Loop N, section] <description>

---

The spec is finalized. You are the final authority — review it before proceeding.

Next steps:
- /test specs/spec-[feature-name].md — generate tests (Red Gate: they should fail)
- Create a plan and run /mob:swarm to implement
```

---

## State File

Maintain `.spec-state.json` in the project root throughout the run. Write it after every phase transition and every loop — this is what allows recovery after context compaction or interruption.

```json
{
  "feature": "<feature-name>",
  "started": "<ISO timestamp>",
  "last_updated": "<ISO timestamp>",
  "status": "interviewing",
  "current_loop": 0,
  "max_loops": 5,
  "requirements_brief": "",
  "spec_path": "specs/spec-<feature-name>.md",
  "deferred": {
    "minor": [
      { "description": "brief description", "section": "8. Edge Cases", "loop": 1 }
    ],
    "suggestions": [
      { "description": "brief description", "section": "10. Testable Properties", "loop": 2 }
    ]
  },
  "resolutions": [
    {
      "issue": "brief description of the finding",
      "original_severity": "major",
      "decision": "what the orchestrator chose to do",
      "reasoning": "why",
      "loop": 1
    }
  ],
  "history": [
    {
      "loop": 1,
      "gate_findings": {
        "critical": 2,
        "major": 3,
        "minor": 1,
        "suggestions": 0,
        "issues": [
          {
            "severity": "critical",
            "title": "Untestable property TP-4",
            "section": "10. Testable Properties",
            "description": "TP-4 asserts 'the system handles errors gracefully' which cannot be falsified"
          }
        ]
      },
      "resolution": "spec-writer-revised",
      "user_input": null
    }
  ]
}
```

Valid `status` values: `"interviewing"`, `"drafting"`, `"critiquing"`, `"complete"`.

Valid `resolution` values: `"spec-writer-revised"`, `"orchestrator-fixed"`, `"user-revised"`, `"accepted-with-issues"`.

---

## Human Escalation (Loop 5)

If Critical or Major issues persist after Loop 5, halt and present:

```
## Spec Blocked — Human Input Required

Feature: [FEATURE NAME]
Loop: 5 of 5

The following issues could not be resolved automatically:

[List all unresolved Critical and Major issues with full context]

The spec has been saved as a draft: [spec path]
State has been saved to .spec-state.json.

To resume after resolving:
  /spec [feature-name] [spec path]

Or provide guidance here and the session will continue.
```

Do not attempt further loops. Wait for the user.

---

## Notes

- Never skip a critique loop, even if the spec feels obviously correct.
- Fix only Critical and Major issues between loops. Minor issues and Suggestions are noted but not addressed.
- The requirements brief is the source of truth for intent. The spec is the source of truth for contract. If they diverge, the spec is wrong.
- Always write to `.spec-state.json` after every loop and phase transition — do not rely on conversational context alone.
- The user is reminded to review at every loop. They are the final authority on requirements — the Spec Gate catches formal defects, not wrong requirements.
- Fresh agents every loop. No context reuse between loops.
