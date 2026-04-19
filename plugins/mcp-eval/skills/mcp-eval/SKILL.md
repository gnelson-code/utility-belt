---
name: mcp-eval
description: |
  Evaluate the attached MCP set by running agent-authored scenarios and producing an adversarial friction report.
  Claude exercises each MCP live, logs every tool call with justifications, then a fresh Sonnet critic audits the transcript.
user-invocable: true
argument-hint: [--name <eval-name>] [--scenarios <file>] [--call-budget <N>] [--wallclock-budget <minutes>]
---

You are the orchestrator of an MCP evaluation session. Your job is to exercise the full set of attached MCPs against user-approved scenarios, record every tool call you make (MCP and non-MCP) with honest justifications, then hand the transcript to the `friction-critic` sub-agent for adversarial review.

You are the primary evaluator because you are the first consumer of these MCPs. The critic exists because you cannot be trusted to grade yourself — you will rationalize your own tool choices. Your job during execution is to try honestly and log what you do. The critic's job is to audit your logs.

**Invariants:**
- Safety dispositions captured in Phase 2 are absolute. A `skip` MCP is never called. A `sandboxed` MCP is never mutated outside the user-provided test resources.
- Every tool call produces exactly one line in `transcript.jsonl`. No call goes unlogged.
- Self-reported `justification` must be written to match what you actually did — not what sounds defensible.

---

## State Schema

`state.json` at `mcp-evals/mcp-eval-<name>/state.json`. Writes are atomic — re-serialize the full object on each transition.

```json
{
  "eval_name": "<name>",
  "started": "<ISO timestamp>",
  "last_updated": "<ISO timestamp>",
  "status": "discovering | interviewing | scoping | scoped | executing | critiquing | reporting | complete",
  "config": {
    "call_budget_per_scenario": 20,
    "wallclock_budget_minutes": 15,
    "scenarios_file": null
  },
  "attached_mcps": [
    {
      "name": "<mcp name>",
      "source": "<config file path>",
      "command": "<command string or null>",
      "tools": ["<tool_name>", "..."]
    }
  ],
  "filtered_mcps": [
    {
      "name": "<mcp name>",
      "reason": "<e.g., 'disabled by .claude/settings.local.json'>"
    }
  ],
  "safety_dispositions": [
    {
      "mcp_name": "<name>",
      "disposition": "read_only | sandboxed | skip",
      "test_resources": [],
      "notes": ""
    }
  ],
  "scenarios": [
    {
      "id": "S1",
      "source": "user | derived",
      "description": "<what the scenario exercises>",
      "targets_mcps": ["<mcp name>"]
    }
  ],
  "execution": [],
  "critic_findings": [],
  "report_path": null
}
```

`critic_findings[]` severity is normalized to title case (`Critical`, `Major`, `Minor`, `Suggestion`) when parsed from the critic's uppercase wire format.

---

## Setup

### Parse Arguments

- `--name <eval-name>`: default is the ISO timestamp slugged as `YYYYMMDD-HHMMSS`.
- `--scenarios <file>`: optional path to a pre-written scenarios file. Format: one scenario per line, or YAML/JSON list with `description` and optional `targets_mcps`.
- `--call-budget <N>`: default `20`. Per-scenario ceiling on tool calls.
- `--wallclock-budget <minutes>`: default `15`. Total wallclock ceiling across all scenarios.

### Check for Existing State

Look for `mcp-evals/mcp-eval-<name>/state.json`. If it exists:

- Validate: must be valid JSON with a `status` field.
- If `status: "complete"`, inform the user the evaluation is finalized and ask whether to start a new one (use a different `--name`) or overwrite.
- If `status` is anything else, present the current phase and ask whether to resume or restart.

If no state exists, create `mcp-evals/mcp-eval-<name>/` and write an initial `state.json` with `status: "discovering"`, timestamps populated, and `config` populated from parsed args.

---

## Phase 1: Discovery

Enumerate every MCP attached to the current Claude Code session. Write them to `attached_mcps[]`.

### Where MCPs Come From

Read these sources and merge, deduplicating by MCP name. For each entry, record the `source` path where you found it.

1. **`~/.claude.json`** — top-level `mcpServers` object, plus per-project entries under `projects[<project-root>].mcpServers`. Match the current session's project root against the `projects[]` keys by longest-prefix match (walk up from the current working directory); do not require exact cwd equality, since `/mcp-eval` may be invoked from a subdirectory.
2. **`.mcp.json`** at the project root — top-level `mcpServers` object. This is a first-class Claude Code convention; if the file exists, read it.
3. **`.claude/settings.json`** and **`.claude/settings.local.json`** — these do not themselves declare MCP servers but may contain three relevant keys that filter which `.mcp.json` servers are active:
   - `enableAllProjectMcpServers: true` — all servers from `.mcp.json` are enabled regardless of the two lists below.
   - `enabledMcpjsonServers: [...]` — explicit allowlist.
   - `disabledMcpjsonServers: [...]` — explicit denylist.
   Apply in that precedence: if `enableAllProjectMcpServers` is true, ignore the allow/deny lists; otherwise a server is active iff it appears in `enabledMcpjsonServers` (or the list is absent/empty — then default is active) and does not appear in `disabledMcpjsonServers`.

Record any MCP that was discovered but filtered out in a `filtered_mcps[]` array in state, with the reason (`"disabled by <settings-path>"`). This makes the final report accurate about what was in scope vs. excluded.

For each MCP, record:
- `name` — the key in `mcpServers`
- `source` — the absolute path of the config file where it was discovered
- `command` — the command string from the config, or `null` for URL-based servers
- `tools` — the list of tool names this MCP exposes (see below)

### Getting Tool Lists

Inspect the current session's tool surface: any tool whose name starts with `mcp__<mcp-name>__` belongs to that MCP. Strip the prefix to get the base tool name. Record what you find; leave `tools: []` for MCPs whose tools cannot be enumerated and note this as a limitation in the eventual report.

### Zero MCPs

If all discovery yields zero entries, print "No MCPs attached — nothing to evaluate" and write `status: "complete"` with an empty `attached_mcps[]`. Write a minimal `report.md` noting the no-op. Do not proceed to the interview phase.

### Transition

When `attached_mcps[]` is populated, update `state.json` with `status: "interviewing"` and the current timestamp.

---

## Phase 2: Safety Interview

Every attached MCP must have a safety disposition before execution can begin. This is the primary defense against destructive side effects.

### Interview Protocol

For each MCP in `attached_mcps[]`, present the user with its name, source, and known tools. Ask:

> For `<mcp name>` — should I treat this as `read_only`, `sandboxed`, or `skip`?
>
> - `read_only`: I will only invoke tools that do not mutate state. If a scenario seems to require mutation, the scenario aborts.
> - `sandboxed`: I may exercise mutating tools, but only against test resources you specify.
> - `skip`: I will not call this MCP at all. Its scenarios will abort with `safety_abort`.

If `sandboxed`, follow up: "Which test resources are approved? List them by identifier (e.g., `repo: eval-playground`, `bucket: test-sandbox`, `path: /tmp/mcp-eval`)." Store the identifiers verbatim in `test_resources[]`.

Do not proceed until every attached MCP has a disposition. An MCP with no disposition is a safety failure, not a default.

### Tool-Level Risk Pre-Check (optional aid)

Before asking, scan each MCP's tool list for names containing `write`, `delete`, `create`, `update`, `push`, `publish`, `send`, `execute`, `run`. Surface these as a hint to the user ("this MCP appears to expose mutating tools — you likely want `sandboxed` or `skip`"). The hint is non-binding; the user's answer is what matters.

### Record

Write each disposition to `safety_dispositions[]`:

```json
{
  "mcp_name": "github",
  "disposition": "sandboxed",
  "test_resources": ["repo: gnelson-code/mcp-eval-playground"],
  "notes": "User confirmed writes are safe against this repo only."
}
```

### Transition

When every MCP has a disposition, update `state.json` with `status: "scoping"`.

---

## Phase 3: Scenario Intake

Scenarios are hybrid: user-provided priorities plus skill-derived edge cases. Both require explicit approval before execution.

### User-Provided Scenarios

If `config.scenarios_file` is set, parse the file. Format is chosen by extension — no content sniffing:

- `.json` → parse as JSON. Must be a list of objects with a required `description` string and optional `targets_mcps` array.
- `.yaml` / `.yml` → parse as YAML. Same shape as JSON.
- `.txt` or any other extension → plain text, one scenario per non-empty line. Lines beginning with `#` are comments and ignored. Leading/trailing whitespace stripped. Plain-text scenarios cannot specify `targets_mcps`; they default to empty and the orchestrator infers target MCPs at execution time.

Validation for each scenario:
- `description` is required and non-empty. If missing or empty, abort with an error naming the offending entry — do not silently drop.
- `targets_mcps` (if present) must be a list of strings that each match a `name` in `attached_mcps[]`. Unknown names abort with an error.
- If a scenario's `targets_mcps` references an MCP whose `safety_dispositions[].disposition` is `skip`, warn the user and ask whether to drop the scenario or change the disposition. If the user does not answer with one of those two options in their next response, default to dropping the scenario and inform them. Do not proceed with a scenario targeting a `skip` MCP under any circumstances.

If no file, interview:

> What scenarios should I exercise? Describe the tasks you want me to try — I'll attempt each one using the attached MCPs and log every call I make.
>
> Examples: "Find all open issues mentioning X and summarize them." "Create a new branch, open a PR, request review." "Search the knowledge base for Y and cross-reference with Z."

Collect 1 to 8 user scenarios. Fewer than 1: nothing to evaluate — stop and tell the user. The 1–8 cap applies to the combined list (user + derived); if the cap is already met by user scenarios, propose zero derived scenarios.

### Derive Edge-Case Scenarios

Inspect the attached tool manifests to propose edge cases. Propose only what the attached surface can meaningfully exercise — no hypotheticals. Filter out MCPs whose disposition is `skip` before considering derivation candidates.

- **Pagination** — if any tool has `limit`, `cursor`, `offset`, `page`, or `per_page` parameters: propose a scenario that walks past the first page.
- **Empty result** — if any tool looks like search/list (`search_*`, `list_*`, `query_*`): propose a query designed to return zero results.
- **Cross-MCP handoff** — only propose when string overlap between two MCPs' tool names is backed by a second signal:
  - **(a) Named-entity parameter match.** One tool has a parameter whose name contains the shared entity noun from the other MCP's tool name — not a generic identifier. Example that qualifies: `github.list_issues` returns issues, and `linear.create_issue` takes a parameter `issue_title` (entity noun "issue" repeated). Example that does NOT qualify: both tools have a parameter named `id` or `url` — these are too generic to imply a shared entity type and must be rejected as a signal.
  - **(b) Explicit user reference.** The user's own scenarios already mention both MCPs by name.
  Pure substring overlap in tool names (e.g., two unrelated `list_users` tools) is never sufficient on its own. If proposed, flag the scenario as `source: "derived"` and note the weak signal at the approval gate — the user must actively accept it.

The auth-failure dimension is intentionally not derived automatically. Auth error shapes surface only when auth fails, and probing auth-failure without breaking auth is unreliable. If the user wants to evaluate auth ergonomics, they should add an explicit scenario.

Derived scenarios are labeled `source: "derived"`. User scenarios are `source: "user"`.

### Approval Gate

Present the full combined list to the user with ids (`S1`, `S2`, ...), source tags, and descriptions. Ask for explicit approval:

> Approve this scenario list? Reply with exactly one of:
> - `approve` — proceed with this list as-is
> - `user-only` — drop all derived scenarios and proceed with user scenarios only
> - An edit request: describe the changes ("drop S3", "rewrite S2 to ...", "add one for ...")
>
> Any other response will be treated as an edit request. Execution does not start until you type `approve` or `user-only`.

Apply this decision mechanically — do not interpret semantically:

```
token = user_response.strip().lower()
if token == "approve":
    status ← proceed with full list
elif token == "user-only":
    status ← drop derived, proceed with user scenarios only
else:
    # Everything else, including "yes", "ok", "sure", "sounds good",
    # "lgtm", "proceed", ":+1:", "approve it", "approve please",
    # or any multi-word response, is an edit request.
    # Do NOT treat these as approval. Ask the user to either describe
    # the specific edit or type exactly "approve" / "user-only".
    status ← pending; re-prompt
```

The match is literal string equality on the trimmed-lowercased token. Anything that is not exactly `approve` or `user-only` — including variations with extra words, punctuation, or synonyms — is an edit request. Re-prompt until the user types one of the two accepted tokens or provides a concrete edit. This rigidity is intentional: the approval gate is the last defense against executing scenarios the user did not fully endorse.

### Transition

Write the final scenario list to `scenarios[]`. Update `state.json` with `status: "scoped"` and halt. The `scoped` status means "scenarios captured, execution not yet begun" — distinct from `executing` (in progress) so future resume logic can tell them apart.

**Halt message** — until Phase 4 of this plan ships, print:

```
Scenarios captured. Execution phase not yet implemented.

Next: Phase 4 adds live execution and structured logging.
Review the scenarios at mcp-evals/mcp-eval-<name>/state.json.
```

Exit without running any scenarios.

---

## Phase 4: Execution

Run each scenario live. Every tool call — MCP or non-MCP — produces exactly one line in `transcript.jsonl`. Budgets cap each scenario independently and cap total wallclock across the session. Safety dispositions are enforced at call time.

### Preconditions

- `state.json` has `status: "scoped"`.
- `safety_dispositions[]` has an entry for every MCP in `attached_mcps[]`.
- `scenarios[]` is non-empty.

If any precondition fails, abort with a clear error. Do not advance state.

Transition `status: "executing"` and record `started` timestamp on the session.

### Transcript File

Open `mcp-evals/mcp-eval-<name>/transcript.jsonl` for append. Each line is one JSON object:

```json
{
  "id": "S1-01",
  "scenario_id": "S1",
  "ts": "<ISO timestamp>",
  "tool": "<tool name>",
  "is_mcp": true,
  "args_summary": "<short redacted summary — do not log raw args with secrets>",
  "result_shape": "<short shape description, e.g. 'array[12] of {id,title,state}'>",
  "justification": null
}
```

Rules for the entry:

- `id` is `<scenario_id>-<counter>` where the counter is zero-padded to 3 digits (`S1-001`, `S1-002`, … `S1-999`). `--call-budget` is clamped at argument-parse time to a maximum of 999.
- `is_mcp` is `true` iff `tool` is a tool name starting with `mcp__`. All other tools (Bash, WebFetch, Read, Write, Grep, Glob, Edit, and any agent spawns) are `is_mcp: false`.
- `justification` is required when `is_mcp: false`. It must (a) name the specific MCP tool the orchestrator considered and rejected, or explicitly state "no attached MCP covers this step"; (b) state in one sentence the reason for the non-MCP tool. Example: `considered mcp__github__search_issues but chose Bash grep because the MCP's search does not support regex and the scenario requires regex matching`. Generic rationalizations ("faster locally", "easier") are not acceptable and will be flagged by the critic. When `is_mcp: true`, `justification` is `null`.
- `args_summary` must not include credentials, tokens, or PII. Summarize shape and a few representative field names; do not dump the full args object. `args_summary` is a logging artifact — it is NOT consulted by safety enforcement (see Safety Enforcement below).
- `result_shape` is a short description of the return shape, not the return body. Example: `array[47] of {id,state,title}` or `object{ok:true,count:47}` or `error:403`. If the call failed, prefix with `error:` and include the status or error class.

### Housekeeping Exemption

Write, Edit, and Read operations whose target path is under `mcp-evals/mcp-eval-<name>/**` (i.e., the eval's own state and transcript files) are **not** logged as tool calls — they are eval bookkeeping, not scenario work. They do not produce transcript entries, do not increment `call_count`, and do not require `justification`. Every other tool call — including Reads of unrelated files, Bash commands, any MCP call — is logged.

### Per-Scenario Loop

For each scenario in order:

1. Record the scenario-start timestamp in `execution[]`:
   ```json
   { "scenario_id": "S1", "status": "running", "call_count": 0, "abort_reason": null, "started": "<ISO>", "ended": null }
   ```
2. Execute the scenario. Attempt it using whatever tools are appropriate — the MCPs, Bash, Read, etc. Behave as you would for any user-assigned task.
3. **Every tool call is two-phase.** This is how the "every call is logged" invariant is enforced. `id` is unique — each logical call produces exactly one line in `transcript.jsonl`, updated in place.
   - **Before dispatch:** run the safety check (see Safety Enforcement) against the *raw args that are about to be sent*. If safety blocks the call, log a `__blocked__` transcript entry (see Safety Enforcement) and do not dispatch. Otherwise, append a transcript entry with `result_shape: null` and increment `call_count`. Then dispatch the call.
   - **After dispatch:** rewrite the same line in place (matched by `id`) so that `result_shape` is filled in with the actual shape. If the call errored, set `result_shape: "error:<class>"`.
   The JSONL file is re-serialized after each update — find the line whose JSON has matching `id` and replace it. Do not append follow-up entries; the invariant is one entry per call, by unique `id`. If the orchestrator crashes mid-dispatch, the file may contain an entry with `result_shape: null` — that is a known crash signature, not a duplicate.
4. After each call, re-check budgets (see below). If either budget is exceeded, abort the scenario.
5. When the scenario's goal is reached or cannot be reached, write `status: "complete"` (or `"aborted"` with `abort_reason`) to its `execution[]` entry, `ended: <ISO>`, and move to the next scenario.

**Known limitation:** the orchestrator is Claude itself. Nothing outside the skill can detect a tool call that was dispatched and not logged. The two-phase discipline makes this unlikely but not impossible. Phase 5's critic cannot detect missing entries — it only sees what was logged. This gap is acknowledged, not closed.

### Budget Enforcement

Two budgets, checked after every tool call within a scenario:

- **Per-scenario call budget.** If `call_count > config.call_budget_per_scenario`, abort the scenario immediately with `abort_reason: "call_budget"`. Write a final transcript entry with `tool: "__budget_abort__"`, `is_mcp: false`, `justification: "call budget exceeded"`.
- **Session wallclock budget.** Compute `now - session_started_ts`. If this exceeds `config.wallclock_budget_minutes`:
  1. Set the current in-flight scenario's `execution[]` entry to `status: "aborted"`, `abort_reason: "wallclock"`, `ended: <trip timestamp>`.
  2. For every remaining un-started scenario, synthesize an `execution[]` entry with `status: "aborted"`, `abort_reason: "wallclock"`, `call_count: 0`, `started` and `ended` both set to the trip timestamp.
  3. Record one final transcript entry with `tool: "__wallclock_abort__"`, `is_mcp: false`, `justification: "session wallclock exceeded"`.

`call_count` on `execution[]` is updated after every successful transcript append — not just at scenario close — so budget and wallclock checks always read a current value.

Do not try to "finish" a scenario that has hit its budget. Budgets are hard.

### Safety Enforcement

The safety check runs **before every dispatch**, against the **raw pre-dispatch args** (the argument object being passed to the tool), never against `args_summary`. `args_summary` is a logging convenience and may omit the values that matter for safety.

Before every tool call targeting an MCP, look up the MCP's disposition in `safety_dispositions[]`. Apply:

**`skip`** — do not dispatch. Log a transcript entry with `is_mcp: false`, `tool: "__blocked__"`, with additional fields `blocked_tool: "<original mcp tool name>"` and `justification: "MCP disposition is 'skip'"`. Safety-blocked calls do **not** increment `call_count` — budgets govern real work, not defensive blocks. The scenario may continue with other tools.

**`read_only`** — classify the call as read-only or potentially mutating using BOTH of these rules against the tool name. Split the name on `_` into lowercase tokens (e.g., `get_or_create_issue` → [`get`, `or`, `create`, `issue`]). Rule matching is against these tokens, never against raw substrings — this avoids false positives like `list_closed_issues` tripping on `close`.
1. **Prefix permit-list.** The first token must be one of: `list`, `get`, `search`, `read`, `show`, `query`, `fetch`, `describe`, `count`, `head`, `inspect`, `peek`, `status`, `exists`, `diff`.
2. **Mutating-token denylist.** No token in the tool name may exactly equal any of: `create`, `write`, `delete`, `update`, `push`, `publish`, `send`, `set`, `add`, `remove`, `replace`, `apply`, `run`, `execute`, `move`, `rename`, `patch`, `insert`, `merge`, `close`, `cancel`, `archive`, `revoke`, `upload`, `ingest`. This catches `get_or_create_issue` (denylist match on `create`) while permitting `get_closed_issues` (no exact `close` token).

A tool passes `read_only` only if rule 1 matches AND rule 2 does not. Manifest description overrides apply in both directions when available: a description explicitly saying "read-only" permits the call even if rule 2 denies; a description mentioning mutation denies the call even if the name passes. If classification cannot be determined, abort with `abort_reason: "safety_abort"` — in doubt means abort. Safety-blocked calls log a `__blocked__` transcript entry and do not increment `call_count`.

**`sandboxed`** — permit mutations, but only against the user-provided `test_resources[]`. Compare the raw args (not `args_summary`) against `test_resources[]`:
- Inspect every value in the args object that is a string.
- A match means: the arg value is exactly equal to a `test_resources[]` entry (case-sensitive), OR the arg value is a substring of a `test_resources[]` entry (i.e., a prefix, suffix, or embedded identifier the user whitelisted). The reverse direction — a test resource being a substring of the arg value — does **not** match; this prevents `eval-playground` in test_resources from permitting a call against `eval-playground-production`.
- If the call is classified as mutating (per the `read_only` rules above) and no arg value matches any `test_resources[]` entry under the above rule, abort with `abort_reason: "safety_abort"`. This is the "no resource identifier present" case — treat as in-doubt, not permitted.
- If the call is classified as a read, permit without the resource-matching check.

Users should list `test_resources[]` entries in the exact form the MCP tools expect. If a tool accepts both `owner/repo` and `repo` forms, list both. The safety check is authoritative. A scenario that would mutate outside its sandbox never runs — there is no override. If the user wants to evaluate a tool against a specific resource, they must update `safety_dispositions[].test_resources` and re-run. Safety-blocked calls do not increment `call_count`.

### Non-MCP Tool Discipline

When reaching for Bash, WebFetch, Python, or any non-MCP tool during a scenario, write the `justification` honestly:

- If an MCP tool could have done this and you chose not to use it, say why. "Faster to grep locally" is honest. "The MCP search is slow" is honest. "No tool exists" when a tool exists is dishonest — the critic will catch it.
- If the non-MCP tool is genuinely outside any MCP's remit (reading a local file unrelated to MCPs, running a local script the user asked for), say so. The critic treats these as expected.

Do not rationalize. The evaluation's value comes from the honesty of this field.

### After All Scenarios

Update state:

- `status: "critiquing"`
- `execution[]` has one entry per scenario with final status
- `session_ended` timestamp recorded

Proceed to Phase 5.

### Execution State Schema

`execution[]` entries:

```json
{
  "scenario_id": "S1",
  "status": "complete | aborted",
  "call_count": 12,
  "abort_reason": null,
  "started": "<ISO>",
  "ended": "<ISO>"
}
```

`abort_reason` values at the `execution[]` (scenario) level: `call_budget`, `wallclock`, `safety_abort`, or `null` for normal completion. `safety_abort` at the scenario level means a mutation outside the sandbox was attempted; individual `__blocked__` transcript entries (skip disposition, read_only denials) are call-level and do not by themselves abort the scenario — the scenario continues with other tools unless a sandbox violation is detected.

### Halt Message

Until Phase 5 of this plan ships, after execution completes print:

```
Execution complete. Critic pass not yet implemented.

Transcript: mcp-evals/mcp-eval-<name>/transcript.jsonl
State: mcp-evals/mcp-eval-<name>/state.json

Next: Phase 5 adds the friction-critic pass and final report.
```

Exit without spawning the critic.

---

## Phase 5: Critic Pass

*Not yet implemented. Shipped in plan Phase 5.*

---

## Phase 6: Report

*Not yet implemented. Shipped in plan Phase 5.*
