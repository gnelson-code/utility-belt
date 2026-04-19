# mcp-eval

Agent-driven evaluation of an attached MCP set. Claude exercises each MCP live against user-approved scenarios, logs every tool call with a self-reported justification, then hands the transcript to a fresh Sonnet critic (`friction-critic`) for adversarial review.

Companion to [mcp-topo](../mcp-topo). Where `mcp-topo` *designs* tool surfaces, `mcp-eval` *grades* them — from an agent's perspective, against real calls, with the authoring agent barred from grading itself.

## Thesis

The first consumer of an MCP is an agent. A human reading the tool list will not notice the ten-line Bash glue the agent reached for when an MCP came up short. A transcript of every call, audited by a separate critic, does.

Self-reported justification fields are treated as **claims to verify**, not ground truth. A capability gap the orchestrator rationalizes away is exactly the friction the critic is spawned to catch.

## Installation

```bash
/plugin marketplace add https://github.com/gnelson-code/utility-belt.git
/plugin install mcp-eval@utility-belt
```

The MCPs under evaluation must be installed and attached to the current Claude Code session — `mcp-eval` evaluates whatever is in scope.

## Usage

```
/mcp-eval [--name <eval-name>] [--scenarios <file>] [--call-budget <N>] [--wallclock-budget <minutes>]
```

| Flag | Default | Purpose |
|------|---------|---------|
| `--name` | ISO timestamp | Eval run identifier. State lives at `mcp-evals/mcp-eval-<name>/`. |
| `--scenarios` | none | Pre-written scenario file (`.json`, `.yaml`, or `.txt`). Interviewed interactively if omitted. |
| `--call-budget` | 20 | Per-scenario ceiling on tool calls. Clamped to 999. |
| `--wallclock-budget` | 15 | Total session wallclock minutes across all scenarios. |

## Phases

1. **Discovery** — Enumerate attached MCPs from `~/.claude.json`, `.mcp.json`, and `.claude/settings*.json`. Apply `enableAllProjectMcpServers` / `enabledMcpjsonServers` / `disabledMcpjsonServers` filters. Record tool lists from the session's `mcp__<name>__*` surface.
2. **Safety Interview** — Every attached MCP gets a disposition: `read_only`, `sandboxed`, or `skip`. `sandboxed` requires user-specified test resources. No disposition means no execution — this is the primary defense against destructive side effects.
3. **Scenario Intake** — User-provided scenarios plus skill-derived edge cases (pagination, empty results, cross-MCP handoffs with strong signals). Combined list capped at 8. Explicit `approve` / `user-only` token required before execution.
4. **Execution** — Each scenario runs live. Every tool call produces exactly one line in `transcript.jsonl` via two-phase logging (pre-dispatch, post-dispatch rewrite). Budgets checked after every call. Safety enforced against raw pre-dispatch args, never against the summary.
5. **Critic Pass** — A fresh `friction-critic` (Sonnet) reads transcript, state, and each MCP's source config, then emits findings along five dimensions. Raw output is persisted before parsing so a mid-parse crash is cheap to resume.
6. **Report** — `report.md` is written with headline counts, per-MCP findings, cross-MCP friction, skipped MCPs, and caveats. Every finding citation is validated in two passes against the transcript; dangling citations surface in Caveats rather than drop the finding.

## Safety Model

Three dispositions, enforced at call time:

- **`read_only`** — tool names must pass both a prefix permit-list (`list`, `get`, `search`, `read`, …) and a mutating-token denylist (`create`, `write`, `delete`, `update`, …). Tokens are split on `_` so `list_closed_issues` doesn't trip on `close` but `get_or_create_issue` does. Manifest description overrides in both directions.
- **`sandboxed`** — mutations permitted only when the raw args contain a string value that matches a user-declared test resource (exact or substring-of-test-resource). The reverse direction does not match, so `eval-playground` in test resources never permits a call against `eval-playground-production`.
- **`skip`** — the MCP is never dispatched. Attempts log a `__blocked__` transcript entry.

Safety-blocked calls don't increment the per-scenario call budget. Budgets govern real work, not defensive blocks. Sandbox violations abort the scenario with `abort_reason: "safety_abort"`.

## Friction Dimensions

The critic audits along exactly five dimensions — no synonyms, no new categories:

- **`glue_code`** — non-MCP calls the orchestrator made when an MCP tool would have covered the step. Critical findings require manifest-level evidence (parameter schema, description), not tool-name matching.
- **`redundant_calls`** — repeated near-identical calls, pagination loops that a larger page would collapse, list-then-get patterns where the list already contained the data.
- **`response_shape`** — non-MCP calls operating on the result of an MCP call (grep/jq/Python munging between MCP hops). Signals that the MCP is returning the wrong shape.
- **`parameter_inference`** — calls whose args the user never specified and the manifest never surfaced; initial failure followed by retries with different args suggesting trial-and-error.
- **`cross_mcp`** — adjacent calls to different MCPs where the output shape of the first didn't match the input of the second.

Severity: **Critical → Major → Minor → Suggestion**, same taxonomy as the rest of utility-belt. If the source config for an MCP is unreadable, `glue_code` findings are capped at Major — name-level evidence still supports Major, but not Critical.

## Artifacts

```
mcp-evals/mcp-eval-<name>/
├── state.json         # phase, config, discovered MCPs, dispositions, scenarios, findings
├── transcript.jsonl   # one line per tool call; two-phase updated in place by id
├── critic-output.md   # raw critic response (full detail prose)
└── report.md          # user-facing report
```

`transcript.jsonl` is the ground truth. `state.json` is the orchestration ledger. `critic-output.md` is preserved verbatim so the parse step is cheap to rerun if it crashes mid-way — the orchestrator's resume logic consults it before re-spawning the critic.

## Design Decisions

**The orchestrator cannot grade itself.** Claude runs the scenarios; a separate Sonnet agent reads the transcript. This is the same correlated-error argument that justifies mob's fresh-context critics — the agent that chose a tool will rationalize that choice.

**Justifications are claims, not facts.** Every non-MCP call during a scenario requires a `justification` string that names the specific MCP tool considered and rejected (or states plainly that no MCP covers the step). "Faster locally" and "easier" are explicitly flagged as weak. The critic verifies against the MCP's source manifest, not the orchestrator's self-report.

**Safety interview is per-MCP, up front, and absolute.** There is no global "dry run" flag. An MCP with no disposition does not get called. A `skip` MCP is never called, full stop. A `sandboxed` MCP is only called against user-declared resources. This shifts the decision to the user before a single tool fires.

**Approval tokens are literal.** The scenario approval gate requires exact `approve` or `user-only`. `"yes"`, `"lgtm"`, `"approve please"` are all treated as edit requests and re-prompt. This rigidity is the last defense against executing scenarios the user didn't fully endorse.

**Two-phase logging.** Every tool call writes a transcript line before dispatch (with `result_shape: null`) and rewrites the same line in place after dispatch. A crash mid-dispatch leaves a `null`-shape entry — a known signature, not a duplicate.

**Derived scenarios require strong signals.** The skill proposes edge cases (pagination, empty results) only when the attached surface can meaningfully exercise them. Cross-MCP scenarios require either a named-entity parameter match or explicit user reference — pure substring overlap in tool names is never sufficient.

**Citation validation doesn't drop findings.** If a finding cites a transcript id that doesn't exist, or cites ids that don't plausibly attribute to the MCP it blames, the finding stays in the report but the failure surfaces in Caveats. The user sees unverifiable claims rather than having them silently deleted.

## Limitations

- The orchestrator is Claude itself. Nothing outside the skill can detect a tool call that was dispatched and not logged. Two-phase discipline makes this unlikely but not impossible; the critic cannot detect missing entries.
- MCPs whose tools cannot be enumerated from the session surface are recorded with `tools: []` and surfaced as a caveat in the report.
- Auth-failure scenarios are not derived automatically. Probing auth-failure without breaking auth is unreliable; if auth ergonomics matter, add an explicit scenario.

## See Also

- [mcp-topo](../mcp-topo) — designs MCP tool surfaces from API analysis
- [probe](../probe) — systematic read-only exploration of databases, REST APIs, and MCPs
