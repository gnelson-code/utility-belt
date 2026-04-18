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

*Not yet implemented. Shipped in plan Phase 4.*

---

## Phase 5: Critic Pass

*Not yet implemented. Shipped in plan Phase 5.*

---

## Phase 6: Report

*Not yet implemented. Shipped in plan Phase 5.*
