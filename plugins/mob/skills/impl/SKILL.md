---
name: impl
description: |
  Lightweight implementation with a single quality critic and progressive de-escalation.
  Implements each phase of a plan with inline verification (tests, linting, coverage) and a quality-gate review.
  First critique uses Opus; subsequent critiques use Sonnet.
user-invocable: true
argument-hint: path/to/plan.md [--from-phase <name or number>]
---

You are the orchestrator of an impl session. Your job is to implement a plan to a standard that satisfies one demanding critic: the Quality Gate. You handle all objective verification (tests, linting, coverage) directly. The critic handles subjective review (logic, edge cases, test quality).

**Important:** Run this skill on the branch where you want the work to land. Impl does not manage branching.

### Severity Levels

- **Critical**: incorrect, unsafe, or non-functional. Must be fixed.
- **Major**: significantly degrades quality, maintainability, or correctness in edge cases. Must be fixed unless downgraded with documented reasoning.
- **Minor**: reduces quality but does not affect correctness. Deferred to final report.
- **Suggestion**: out of scope. Recorded in final report only.

---

## Setup

### Parse Arguments

- First argument: path to the plan file (required)
- `--from-phase <name or number>`: start from a specific phase, skipping all prior phases

### Read the Plan

Read the plan file in full. Parse it into an ordered list of phases. If the structure is ambiguous, parse using your judgment.

### Detect the Toolchain

Examine the project root and identify the language, test runner, linter, and coverage tool. Do this once — reuse for all phases.

| Signal | Language | Test runner | Linter | Coverage |
|--------|----------|-------------|--------|----------|
| `pyproject.toml`, `setup.py`, `requirements.txt` | Python | pytest | ruff or flake8 | `pytest --cov` |
| `package.json` only | JavaScript | per `scripts` (jest, vitest, mocha) | eslint | per config |
| `tsconfig.json` + `package.json` | TypeScript | per `scripts` | eslint + tsc | per config |
| `Cargo.toml` | Rust | `cargo test` | `cargo clippy` | `cargo llvm-cov` |
| `go.mod` | Go | `go test ./...` | `go vet` + staticcheck | `go test -cover` |
| `Makefile` present | Any | `make test` | `make lint` | `make coverage` |

Use the scripts defined in the project's config files rather than invoking tools directly — the project may have configured them with important flags.

**Polyglot projects:** Detect all languages present. Run tools for each language that has files in the set of modified files.

**No toolchain detected:** Skip the verification step's tool execution. Note this as a finding (Minor if early-stage, Major if the code is complex). The quality-gate critique still runs on the code itself.

### Determine Starting Phase

- If `--from-phase` is specified, begin at that phase (by name match or 1-based index)
- Otherwise, begin at phase 1

---

## Per-Phase Loop

Execute the following for each phase. A **loop** is one cycle of: implement → verify → critique → triage. Maximum 3 loops per phase.

### Step 1: Implement

**Loop 1:** Implement everything described in the phase.
- Read surrounding code before writing — follow codebase conventions
- Write tests alongside the implementation
- Record the list of files created or modified

**Loops 2–3:** Fix only Critical and Major issues from the previous loop.
- Address each issue directly and completely
- Do not make unrelated changes
- Do not address Minor issues or Suggestions

### Step 2: Verify

Run the project's toolchain directly. This is not delegated to an agent — you execute these yourself.

#### 2a. Run Tests

Run the full test suite. Capture all output.

- **All tests pass:** Proceed. Record "tests: pass" for this loop.
- **Tests fail:**
  - If failures are caused by the new code → **Critical**. These must be fixed.
  - If failures are pre-existing (present before this phase) → note but do not block.
  - Quote the failure output verbatim — you will pass it to the quality-gate.

#### 2b. Run Linter

Run the project's linter. Capture all output.

- **Lint errors** (not warnings) on new/modified files → **Major**.
- **Lint warnings** on new/modified files → **Minor**.
- Quote lint output verbatim.

#### 2c. Run Coverage

Run coverage on the test suite if the toolchain supports it. Capture output.

- **Critical code paths with no coverage** → **Major**.
- Do not require 100% coverage — focus on paths that matter (error handling, core logic, boundary conditions).
- Quote coverage output verbatim.

#### 2d. Compile Verification Results

Assemble the verification results into a structured block:

```
## Verification Results

### Tests
[PASS/FAIL — N passed, N failed]
[If failures: quote the relevant output]

### Linter
[CLEAN/ISSUES — quote any errors or warnings on new files]

### Coverage
[summary — quote relevant output]
```

This block is passed to the quality-gate in the next step.

### Step 3: Quality-Gate Critique

Spawn the `quality-gate` agent as a fresh sub-agent. **Do not reuse agents between loops or phases.**

**Model selection** (use the Agent tool's `model` parameter, which overrides agent frontmatter):
- **Loop 1:** Spawn with `model: opus`
- **Loops 2–3:** Spawn with `model: sonnet`

**`quality-gate` prompt:**
```
Phase: [PHASE NAME]
Goal: [copy the phase description from the plan verbatim]
Loop: [N] of 3

The following files were created or modified:
[list each file with its full path]

## Verification Results (already executed — do not re-run these tools)

[paste the verification results block from Step 2d]

The test suite, linter, and coverage have already been run. The results above are authoritative. Do NOT re-run tests, linting, or coverage.

Focus your review on what tools cannot catch:
- Logic correctness — does the code actually do what the phase goal describes?
- Edge case handling — what happens with empty inputs, boundary conditions, unexpected state?
- Test skepticism — do the tests actually exercise the logic? Could you delete the core logic and have them still pass?
- Clarity — could a new engineer understand and modify this without introducing bugs?

Use the severity classification in your instructions.
```

Wait for the agent to complete.

### Step 4: Triage

Combine the verification results (Step 2) with the critic output (Step 3). Classify every issue:

- Count Critical issues (from verification + critic)
- Count Major issues (from verification + critic)
- Note Minor issues and Suggestions (deferred to report)

When the critic flags something at Critical or Major, you may investigate and downgrade if the evidence supports it. Document any downgrade with your reasoning.

**Decision:**

| Condition | Action |
|-----------|--------|
| Zero Critical, zero Major | Phase complete. Commit, move to next phase. |
| Critical or Major remain, this is Loop 1 or 2 | Fix issues. Begin next loop. |
| Critical or Major remain, this is Loop 3 | Stop. Request human input. |

### Committing

After a phase completes (zero Critical/Major), commit:

```bash
git add <files modified in this phase>
git commit -m "impl: <phase name>"
```

Do not commit between loops — only after the phase passes.

**If the commit fails** (pre-commit hooks, formatting, etc.): read the error, fix, and attempt once more. If it fails again, stop and present the failure to the user.

### Human Escalation (Loop 3 Failure)

If Critical or Major issues persist after Loop 3:

```
## Impl Blocked — Human Input Required

Phase: [PHASE NAME]
Loop: 3 of 3

The following issues could not be resolved:

[List all unresolved Critical and Major issues with full context]

To resume after resolving:
  /mob:impl path/to/plan.md --from-phase "[PHASE NAME]"

Or provide guidance here.
```

Do not attempt further loops. Wait for the user.

---

## Final Report

After all phases complete (or upon escalation), produce two outputs:

### 1. Write `impl-report.md` to the project root

```markdown
# Impl Report

**Plan:** [plan file path]
**Date:** [date]
**Branch:** [current git branch]

## Summary

[1-2 sentence overall assessment — phases completed, total loops, any escalations]

## Phases

### Phase 1: [Name]

**Status:** Complete
**Loops:** [N]

**Verification:**
- Tests: [pass/fail summary]
- Linter: [clean/issues summary]
- Coverage: [summary]

**Quality Gate (Loop 1, Opus):**
- [summary of findings]
- Issues fixed: [list]

**Quality Gate (Loop 2, Sonnet):** *(if applicable)*
- [summary of findings]
- Issues fixed: [list]

---

### Phase N: [Name] *(Escalated)*

**Status:** Escalated at Loop 3
**Unresolved Issues:**
- [CRITICAL] ...
- [MAJOR] ...

---

## Deferred Items

### Minor Issues
- [Phase N, Loop N, source] <description>

### Suggestions
- [Phase N, Loop N, source] <description>
```

### 2. Print a Terminal Summary

```
## Impl Complete

Plan: [path]
Phases: [completed]/[total]
Total loops: [N]
[Any unresolved issues]

Report: impl-report.md

[If all phases passed:]
Ready for PR.
```

---

## Notes

- Never skip verification, even if the implementation feels obviously correct.
- Fix only Critical and Major issues between loops. Minor and Suggestions are deferred.
- The quality-gate agent must NOT re-run tests, linting, or coverage. Those results are passed in. The agent reviews code, not tool output freshness.
- Progressive de-escalation is deliberate: Opus catches the deep issues on the first pass; Sonnet handles residual cleanup efficiently.
- If a fix introduces new problems, the next loop's verification will catch them. Implement cleanly.
