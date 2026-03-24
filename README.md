# utility-belt

A suite of Claude Code plugins for structured, adversarial AI-assisted software delivery.

Ask Claude to write a feature and you'll get something that works. But single-pass generation has a quality ceiling — the agent that wrote the code can't reliably catch its own mistakes. These plugins break that ceiling with **formal specifications, adversarial critic loops, and multi-agent orchestration** where the agent that wrote the code is never the agent that reviews it.

These plugins are open-source distillations of patterns I've been using in production since October 2025 to ship MCPs, NER/entity-linking and other NLP systems, chatbots, and knowledge graph features with AI-assisted workflows.

**They are starting points.** Download them, read them, customize them. The agent definitions, severity thresholds, loop caps, and orchestration logic are all in markdown — change what doesn't fit your workflow.

## Installation

```bash
# Add the marketplace
/plugin marketplace add grahamnelson/utility-belt

# Install individual plugins
/plugin install mob@utility-belt
/plugin install probe@utility-belt
/plugin install mcp-topo@utility-belt
```

## Model Configuration

All agents default to Sonnet, Opus, or Haiku via Claude Code's standard model routing. I strongly recommend **substituting a peer model from a different provider for Opus where possible** — using the same model family for both implementation and critique reintroduces the correlated-error problem the architecture is designed to prevent. 

As always, I highly recommend customizing them to suit your needs, or building off of them to find something that works for you.

## Plugins

### mob

Adversarial mob programming for large features. Designed for work that resembles epics more than tickets — multi-phase delivery where quality compounding matters.

This is implementation based on the idea of having a team of engineers at your fingertips: different personas, priorities, and roles that come together to ship something.

**You don't need to be an engineer to use this, and in fact product or design folks may get more out of some steps than engineers.** `/mob:spec` and `/mob:test` are built for anyone who can describe what a feature should do. A product manager can produce a formal specification and a failing test suite without writing code — then hand both to an engineer (or to `/mob:swarm`) for implementation. The spec and test stages are force multipliers for people who know what "done" looks like but don't ship code themselves. 

The full pipeline: **spec → test → plan → implement → PR**, with adversarial gates at each stage. Use `/mob:sprint` for epics that need the full pipeline. Use `/mob:swarm` directly when you already have a plan and just need adversarial implementation — sufficient for large tickets.

Generally, I recommend being careful with `/mob:sprint`. You'll burn tokens and get a good result, but many tasks may not require it. For most things `/mob:swarm` is good enough. Product designers and less technical individuals with clear vision may get the most out of `/mob:sprint`. 

| Command | Description |
|---------|-------------|
| `/mob:sprint` | End-to-end epic delivery. Orchestrates spec, test, plan, and implementation into a single autonomous pipeline. |
| `/mob:swarm` | Phased implementation with adversarial critique. Takes an existing plan and executes it — Architecture Gate and Quality Gate critique each phase in parallel. Critical/Major issues trigger fix loops (max 3), then escalate to the human. |
| `/mob:spec` | Produces a formal feature specification through structured interview, drafting, and adversarial critique. |
| `/mob:test` | Generates a failing test suite from a spec. The Red Gate validates that tests fail correctly and cover the spec's testable properties. |

**Seven specialized agents** handle writing and critique. Writers produce artifacts (specs, tests, plans, code). Gates tear them apart — each spawned fresh with no shared context, so they can't inherit the writer's blind spots.

| Role | Writers | Gates |
|------|---------|-------|
| Specification | spec-writer | spec-gate |
| Tests | test-writer | red-gate |
| Plan | plan-writer | *(none — plans require human judgment)* |
| Implementation | orchestrator | architecture-gate, quality-gate |

All gates classify findings as **Critical → Major → Minor → Suggestion**. Critical and Major trigger fix loops. Minor and Suggestion are deferred. The same taxonomy across every gate is what makes automated triage work.

### probe

Systematic exploration of unfamiliar interfaces using only read-only operations. Produces structured capability profiles.

| Command | Description |
|---------|-------------|
| `/probe` | Explore a database, REST API, or MCP server. Outputs a capability profile, raw findings, and surface map. |

**Supported interfaces:** PostgreSQL/MySQL databases, REST APIs (with OpenAPI spec or endpoint discovery), MCP servers.

Probe operates in five phases:

1. **Connect & Preflight** — Establish connection, assess safety enforcement (Strong/Moderate/Weak).
2. **Enumerate Surface** — List tables, endpoints, or tools with metadata.
3. **Deep Probe** — Test behaviors, edge cases, error handling, and consistency. Read-only throughout.
4. **Synthesize** — Organize findings by functional area into a coherent capability profile.
5. **Output** — Write `profile.md`, `raw-findings.json`, and `surface-map.md` to `probes/probe-<name>/`.

Supports `--diff` for delta analysis against a previous probe and `--mode human|machine` for output format.

### mcp-topo

MCP topo, short for "MCP Topology", designs MCP server tool surfaces by analyzing an API and proposing interaction categories rather than 1:1 endpoint mappings. 

This plugin is based on the idea that the first consumers of MCP interfaces are not people - they are agents and _secondarily_ people. I've noticed that this can be fairly challenging to think through or adapt to, so this skill is meant to formalize that process. 

This skill is aimed at anyone interested in making MCP design decisions, technical or otherwise. Product managers in particular may benefit. This plugin is extremely opinionated and creates methodically formed recommendations of agent-optimal MCP schemas. 

| Command | Description |
|---------|-------------|
| `/mcp-topo` | Analyze an API (OpenAPI spec, codebase, or URL) and design a tool surface. Outputs a plan consumable by `/mob:swarm`. |

The plugin's core argument: mapping every API endpoint to its own MCP tool produces a tool surface that's hard to use and hard to maintain. Instead, mcp-topo groups endpoints into **interaction categories** based on consumer intent, then stress-tests the design with a devil's advocate agent that argues for 1:1 mapping.

Five phases: **Intake → Scope Narrowing → Surface Analysis → Hypothesis (with devil's advocate) → Plan Output**.

## What a Mob Session Looks Like

Running `/mob:sprint add-webhook-support` produces roughly this sequence:

1. **Spec** — Interactive interview → formal specification with numbered behavioral definitions, preconditions, invariants, and edge cases → spec-gate critique loop until zero Critical/Major findings
2. **Test** — Test suite generated from spec with traceability (`test_TP1_webhook_delivery`, `test_ERR2_invalid_signature`) → red-gate verifies every test fails (no implementation yet) and catches weak assertions
3. **Plan** — Phased implementation plan with ≤8 files per phase, mapped to specific tests → human review (no automated gate)
4. **Implement** — Each phase built, then architecture-gate and quality-gate critique in parallel → fix loops on Critical/Major → commit and advance
5. **PR** — Draft pull request with links to spec, plan, and deferred items

The whole pipeline runs autonomously by default. Use `--checkpoints` to pause between stages.

## Design Decisions

**Adversarial critics use fresh context.** Each critic agent is spawned fresh — no shared context with the implementation agent. This prevents correlated errors. If the implementer misunderstood a requirement, a critic sharing that context would likely miss the same thing.

**Bounded loops with human escalation.** Critic loops cap at 3 iterations (spec-gate caps at 5). If issues persist after the cap, the system escalates to the human rather than looping indefinitely. Infinite loops are a failure mode, not a feature.

**No plan-gate.** The spec has a gate. The tests have a gate. The implementation has two gates. The plan doesn't. Plans require human judgment about decomposition, scope, and sequencing that no critic agent has sufficient context to evaluate.

**State recovery across context compaction.** Every skill maintains a `.state.json` file tracking phase, loop count, findings, and outputs. If Claude Code's context window compresses mid-run, the skill can recover from the state file rather than restarting.

**Consistent severity taxonomy.** All critics classify findings as Critical, Major, Minor, or Suggestion. This consistency is what allows the orchestrators to make automated triage decisions — the same severity means the same thing regardless of which critic produced it.

**Test-before-implementation.** The pipeline writes tests from the spec before any code exists, then validates they all fail. This catches tautological assertions and ensures tests are actually testing the spec, not retroactively confirming whatever the implementation happened to produce.

## Under the Hood

The plugin definitions are markdown — readable and forkable. Here's what's actually in them.

### mob internals

**Parallel critic dispatch.** During implementation, the orchestrator spawns architecture-gate and quality-gate simultaneously. Both complete before triage begins — neither critic sees the other's findings, preventing groupthink.

**Orchestrator downgrade authority.** When only one of two critics flags an issue, the orchestrator can investigate and downgrade the severity — but must document its reasoning in the state file. This prevents a single overzealous critic from blocking progress while maintaining an audit trail.

**Respawn threshold.** If the red-gate finds >50% of tests have Critical/Major issues, it doesn't loop — it respawns the entire test-writer agent with a fresh context. Patching a fundamentally broken test suite is worse than regenerating it.

**Traceability matrices.** Every test traces to specific spec items (`test_TP1_webhook_delivery` → TP-1 → B-3, EDGE-2). Coverage tracking maps every testable property, error case, invariant, precondition, and edge case to its test — gaps are visible by inspection.

**Spec formalism.** Specifications use RFC 2119 keywords (MUST/MUST NOT/MAY) and distinguish between behavioral definitions, preconditions, postconditions, invariants, and edge cases. The spec-gate rejects unfalsifiable properties — "the system should be fast" doesn't survive critique.

### probe internals

**Three-layer read-only enforcement.** Safety isn't a flag — it's enforced at the infrastructure level (database privileges, read-only API scopes), the structural level (read-only transactions, MCP tool classification by keyword analysis), and the statement level (no mutation operations issued). All three must hold.

**Engine-specific hazard avoidance.** Probe knows that Neo4j's unbounded traversal (`[*]`) can bring down a graph database, that MongoDB's `countDocuments({})` is a full collection scan while `estimatedDocumentCount()` is not, and that DynamoDB `Scan` should never be used when `Query` with a partition key is available. I've attempted to avoid generic safety rules and include per-engine constraints where I am familiar with them. 

**Cost-aware discovery.** For large tables (>100k rows), probe uses statistics tables instead of `COUNT(*)` and applies `TABLESAMPLE` where available. It retrieves full `DISTINCT` sets only for low-cardinality columns (<50 values). It warns when >100 items are in scope before token costs escalate.

**Rate limit detection.** When probe hits a rate limit, it stops immediately — no retry logic. It saves state with `blocked_reason: "rate_limit"`, surfaces the `X-RateLimit-*` headers, and lets the human decide. Automated retry against someone else's API is not probe's call to make.

**Delta mode.** `--diff` matches findings by surface ID and category against a previous probe, classifying changes as Added/Removed/Changed. Useful for tracking API drift or database schema evolution over time.

### mcp-topo internals

**Devil's advocate agent.** Before the user sees any proposed tool surface, a dedicated adversary evaluates each category across five dimensions: independence, safety/reversibility, parameter schema coherence, cognitive load, and composability loss. It steel-mans the case for 1:1 endpoint mapping with API-specific arguments, not generic ones.

**CRUD detection as failure signal.** If the proposed categories look like CRUD-by-entity ("create_repo", "list_repos", "update_repo"), the analysis has failed — mcp-topo restarts from consumer tasks. Categories are named by intent ("manage_repository_lifecycle", "review_code_changes"), not by entity or operation.

**Coverage verification.** A coverage check detects unassigned capabilities, double-assigned capabilities, and phantom references. It must report "COVERAGE OK" before the devil's advocate runs. If categories change after synthesis, the check runs again.

**Large API handling.** APIs with >30 endpoints trigger automatic scope narrowing — domain extraction from OpenAPI tags, path prefixes, or codebase directory structure, presented as a domain-task matrix for informed scoping before analysis begins. For 100+ capabilities, state files store abbreviated entries and reference a separate capabilities file to keep state manageable.

**Anti-pattern comparison.** Every design is presented alongside what a 1:1 mapping would produce — real tool count, redundant parameter passing, context fragmentation, and token cost. The comparison honestly acknowledges where the devil's advocate identified legitimate splits.

## Repository Structure

```
utility-belt/
├── plugins/
│   ├── mob/
│   │   ├── agents/          # Specialist agent definitions
│   │   ├── commands/         # User-invocable command dispatchers
│   │   └── skills/           # Orchestration logic and state management
│   ├── probe/
│   │   ├── commands/
│   │   └── skills/
│   └── mcp-topo/
│       ├── agents/
│       ├── commands/
│       └── skills/
└── .claude-plugin/
    └── marketplace.json      # Plugin registry
```

## Acknowledgments

These plugins were inspired by and build on ideas from:

- [**VSDD (Verified Spec-Driven Development)**](https://gist.github.com/dollspace-gay/d8d3bc3ecf4188df049d7a4726bb2a00) by dollspace-gay — VSDD's formalization of spec → test → implement pipelines with adversarial verification influenced the ordering and verification philosophy behind mob.
- [**ed3d-plugins**](https://github.com/ed3dai/ed3d-plugins) by ed3dai — Claude Code plugins implementing a research-plan-implement workflow
- [**learning-opportunities**](https://github.com/DrCatHicks/learning-opportunities) by Dr. Cat Hicks — a Claude Code skill for deliberate skill development during AI-assisted coding

## Author

Graham Nelson — Lead MLE

[LinkedIn](https://www.linkedin.com/in/grahamenelson/) · [GitHub](https://github.com/grahamnelson) · graham.nelson94@gmail.com

## License

MIT
