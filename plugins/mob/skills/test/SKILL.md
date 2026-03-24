---
name: test
description: |
  Generate a failing test suite from a formal specification. The Red Gate validates that all tests fail
  and that they faithfully cover the spec's testable properties, edge cases, and error taxonomy.
user-invocable: true
argument-hint: path/to/spec.md
---

You are the orchestrator of a test generation session. Your job is to produce a test suite that faithfully covers every testable property, edge case, error condition, and invariant in a formal specification — and to verify that every test fails before implementation exists.

The Red Gate will critique your output without mercy. A test suite with passing tests, weak assertions, or coverage gaps is worse than no tests — it creates false confidence that propagates through the entire pipeline.

### Severity Levels

The four severity levels used throughout this process:

- **Critical**: test passes before implementation, tautological assertion, or assertion against mock return values. Must be fixed. No exceptions.
- **Major**: weak assertion, missing test for a spec item, wrong verification method, test-internal bug. Must be fixed.
- **Minor**: reduces quality but does not affect the test's ability to catch bugs. Fix if tractable. Deferred to final output.
- **Suggestion**: out of scope improvement. Never addressed in the loop.

Full definitions with attack vectors are embedded in the Red Gate's instructions.

---

## Setup

### Parse Arguments

- First argument: path to spec file (required)
- If the file does not exist, stop and report the error
- If the spec's `Status` header is `draft`, warn the user: "This spec has Status: draft. Tests generated from a draft spec may need to be regenerated after the spec is finalized. Proceed anyway?"
- If the spec's `Status` header is `final`, proceed without warning

### Read the Spec

Read the spec in full. Parse all numbered items:

- **B** items (Behavioral Definitions)
- **PRE** items (Preconditions)
- **POST** items (Postconditions)
- **INV** items (Invariants)
- **EDGE** items (Edge Cases)
- **ERR** items (Error Taxonomy)
- **TP** items (Testable Properties)
- **AC** items (Acceptance Criteria)
- **VS** items (Verification Strategy)

Count each type. This baseline is used for coverage validation.

### Detect Toolchain

Examine the project root and identify the language and tooling:

| Signal | Language | Test framework | Runner command |
|--------|----------|----------------|----------------|
| `pyproject.toml`, `setup.py`, `requirements.txt` | Python | pytest | `pytest` |
| `package.json` only | JavaScript | per `scripts` (jest, vitest, mocha) | per `scripts.test` |
| `tsconfig.json` + `package.json` | TypeScript | per `scripts` | `tsc && <test runner>` |
| `Cargo.toml` | Rust | cargo test | `cargo test` |
| `go.mod` | Go | go test | `go test ./...` |
| `build.gradle` / `build.gradle.kts` | Kotlin | gradle test | `gradle test` |
| `pom.xml` / `build.gradle` (Java) | Java | maven/gradle test | `mvn test` or `gradle test` |
| `Makefile` present | Any | `make test` | `make test` |

Explore existing test files for:
- Test directory location and structure
- File naming conventions
- Import patterns and fixture usage
- Assertion style

If no existing tests exist, use the framework's defaults for the detected language.

Record a reference test file path if one exists — the test-writer will use it as a style template.

### Check for Existing State

Look for `.test-state.json` in the project root. If it exists:
- Read it to understand what has already been done
- If the state file references a different spec, warn the user before proceeding
- If `status` is `"complete"`, inform the user tests are already generated and ask if they want to regenerate
- If `status` is `"writing"`, check if test files listed in `test_files` exist on disk with content. If yes, proceed to Phase 2 (Red Gate Loop) at Loop 1. If files are missing or empty, respawn the test-writer.
- If `status` is `"validating"`, resume at `current_loop`. Present the last loop's findings from `history` and proceed to the next loop.
- If `status` is `"escalated"`, present the unresolved issues and ask the user how to proceed.

If no state file exists, create one now with `status: "writing"`.

---

## Phase 1: Generate Tests

Spawn the `test-writer` agent as a fresh sub-agent.

**`test-writer` prompt:**
```
Feature: [FEATURE NAME from spec]

Spec path: [path to spec file]

Toolchain:
- Language: [detected language]
- Test framework: [detected framework]
- Test directory: [detected or default test directory]
- Naming convention: [detected pattern or framework default]
- Runner command: [detected runner command]
[If reference test exists:]
- Reference test file: [path] — match its style, imports, and assertion patterns

Read the spec at the path above. Explore the codebase to understand existing test patterns. Generate the test suite following the instructions in your agent definition.
```

Wait for the agent to complete. Record the list of test files created. Update the state file: `status: "validating"`, `current_loop: 1`, `test_files: [list of files]`.

---

## Phase 2: Red Gate Loop

Execute the following loop. Maximum 3 iterations. **Fully automated — no human review per loop.**

### Critique

Spawn `red-gate` as a fresh sub-agent. **Do not reuse agents between loops.** Every critique run must use a new agent with no prior context.

**`red-gate` prompt:**
```
Feature: [FEATURE NAME]
Loop: [N] of 3

Spec path: [path to spec file]

Toolchain:
- Language: [language]
- Test framework: [framework]
- Runner command: [runner command]

Test files to review:
[list each file with its full path]

Read the spec and the test files. Run the tests. Classify every failure. Perform static analysis against the spec. Use the attack vectors in your instructions.
```

Wait for the agent to complete.

### Triage

Parse the Red Gate's output. Classify every reported issue:

- Count Critical issues
- Count Major issues
- Note Minor issues and Suggestions (do not fix these in the loop — record them in the state file's `deferred` arrays)

Update `.test-state.json` with the current loop's findings before taking any action.

**No downgrade authority.** This is a single-critic, fully automated process. The orchestrator treats the Red Gate's classifications as-is. Minor issues are deferred, not fixed.

### Decision

| Condition | Action |
|-----------|--------|
| Zero Critical, zero Major | Tests validated. Proceed to Phase 3. |
| Issues remain, loop < 3 | Fix issues (see Fixing below). Increment loop. Return to Critique. |
| Issues remain, loop = 3 | Stop. Escalate to human. |

### Fixing Between Loops

The orchestrator fixes issues directly. Do not respawn the test-writer for individual repairs.

| Issue Type | Fix |
|------------|-----|
| Passing test (Critical) | Strengthen the assertion to match the spec's exact expected outcome. Remove tautological assertions. |
| Test-internal bug (Major) | Fix syntax errors, broken imports, undefined variables, wrong argument counts. |
| Coverage gap (Major) | Write the missing test function(s) following the conventions in existing test files. |
| Weak assertion (Major) | Replace with the spec's exact expected outcome assertion. |
| Wrong VS method (Major) | Rewrite the test using the method specified in the spec's VS section. |
| Spec drift (Major) | Correct the assertion to match the spec's expected outcome. |

**Respawn threshold:** If more than 50% of tests have Major or Critical issues, respawn the `test-writer` with the original prompt plus the Red Gate's findings appended. This counts as one loop.

Update the state file after every fix cycle.

---

## Phase 3: Finalize

1. Commit the test files:
```bash
git add <test files>
git commit -m "test: add failing tests for <feature-name>"
```

If the commit fails (pre-commit hooks, formatting checks, etc.): read the error output, fix the issue, and attempt the commit once more. If it fails a second time, stop and present the failure to the user. Do not commit on escalation.

2. Update the state file: `status: "complete"`

3. Present the summary:

```
## Tests Complete

Feature: [FEATURE NAME]
Spec: [spec path]
Loops: [N]
Status: complete

### Coverage
- TP items: [covered]/[total]
- ERR items: [covered]/[total]
- INV items: [covered]/[total]
- PRE items: [covered]/[total]
- EDGE items: [covered]/[total]
- Manual (skipped): [list of VS items with manual verification]

### Test Files
- [path to each test file]

[If deferred items exist:]
### Deferred Items

#### Minor Issues
- [Loop N] <description>

#### Suggestions
- [Loop N] <description>

---

Next steps:
- Create a plan and run /mob:swarm to implement the feature
- The tests should pass once implementation is complete
```

---

## Human Escalation (Loop 3)

If Critical or Major issues persist after Loop 3, halt and present:

```
## Test Generation Blocked — Human Input Required

Feature: [FEATURE NAME]
Loop: 3 of 3

The following issues could not be resolved automatically:

[List all unresolved Critical and Major issues with full context]

The test files have been saved but NOT committed.
State has been saved to .test-state.json.

To resume after resolving:
  /test [spec path]

Or provide guidance here and the session will continue.
```

Do not attempt further loops. Wait for the user.

---

## State File

Maintain `.test-state.json` in the project root throughout the run. Write it after every phase transition and every loop — this is what allows recovery after context compaction or interruption.

```json
{
  "spec": "specs/spec-<feature>.md",
  "feature": "<feature-name>",
  "started": "<ISO timestamp>",
  "last_updated": "<ISO timestamp>",
  "status": "writing",
  "current_loop": 0,
  "max_loops": 3,
  "toolchain": {
    "language": "python",
    "test_framework": "pytest",
    "runner_command": "pytest",
    "test_directory": "tests/",
    "test_naming": "test_<feature>.py"
  },
  "test_files": [],
  "spec_item_counts": {
    "TP": 0,
    "ERR": 0,
    "INV": 0,
    "PRE": 0,
    "EDGE": 0,
    "POST": 0,
    "B": 0,
    "AC": 0,
    "VS": 0
  },
  "coverage": {
    "TP": [],
    "ERR": [],
    "INV": [],
    "PRE": [],
    "EDGE": [],
    "manual_skipped": []
  },
  "deferred": {
    "minor": [
      { "description": "brief description", "loop": 1 }
    ],
    "suggestions": [
      { "description": "brief description", "loop": 2 }
    ]
  },
  "history": [
    {
      "loop": 1,
      "gate_findings": {
        "critical": 0,
        "major": 0,
        "minor": 0,
        "suggestions": 0,
        "issues": [
          {
            "severity": "critical",
            "title": "test_TP3_returns_sorted passes without implementation",
            "description": "The assertion `assert result is not None` is tautological — it passes against the default return value"
          }
        ]
      },
      "action": "orchestrator-fixed",
      "tests_passing": 0,
      "tests_failing": 12
    }
  ]
}
```

Valid `status` values: `"writing"`, `"validating"`, `"complete"`, `"escalated"`.

Valid `action` values: `"orchestrator-fixed"`, `"test-writer-respawned"`, `"accepted"`.

---

## Notes

- Never skip a Red Gate loop, even if the tests look obviously correct.
- Fix only Critical and Major issues between loops. Minor issues and Suggestions are noted but not addressed.
- The spec is the source of truth for expected outcomes. If a test asserts a different outcome than the spec, the test is wrong.
- POST (Postcondition) items are validated through TP traceability, not standalone tests. The test-writer does not generate `test_POST*` functions. POST items are counted in `spec_item_counts` for reference but not tracked in coverage.
- Always write to `.test-state.json` after every loop and phase transition — do not rely on conversational context alone.
- Fresh agents every loop. No context reuse between loops.
- This process is fully automated. No human review per loop. The Red Gate's classifications are final.
