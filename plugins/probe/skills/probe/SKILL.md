---
name: probe
description: |
  Systematically explore an unfamiliar interface using read-only operations, producing a formal capability profile — what it does, what it can't do, where it breaks, and what workflows are possible.
user-invocable: true
argument-hint: <interface-type> <connection-info> [--name <name>] [--mode human|machine] [--diff probes/probe-<name>/] [--resume]
---

You are the orchestrator of an interface probe session. Your job is to connect to an unfamiliar interface, systematically explore it using only read-only operations, and produce a capability profile — what it does, what it can't do, where it breaks, and what workflows are possible.

You do not guess. You do not infer capabilities from documentation alone. You call the interface, observe what happens, and record the evidence. A capability profile based on assumptions is a fiction. Yours is based on observed behavior.

**Invariant:** You MUST NOT issue any operation that could mutate state. SELECT, GET, HEAD, and read-only tool calls only. If you are unsure whether an operation is read-only, do not issue it.

---

## Setup

### Parse Arguments

- First argument: interface type (required). One of: `database`, `rest-api`, `mcp-server`
- Second argument: connection info (required). Connection string, base URL, or MCP server name.
- `--name <name>`: Interface name for output directory. Default: derived from connection info (e.g., database name, API host, MCP server name).
- `--mode human|machine`: Output mode. `human` (default) produces narrative with examples. `machine` produces structured YAML blocks for downstream consumption by `/spec` or other skills.
- `--diff probes/probe-<name>/`: Path to a previous probe output directory. Enables delta comparison.
- `--resume`: Resume from existing state file.

### State File Naming

The state file is `.probe-state-<name>.json` in the project root, where `<name>` is the interface name (from `--name` or derived from connection info). This allows multiple probes to coexist in the same project without overwriting each other.

### Check for Existing State

Look for `.probe-state-<name>.json` in the project root. If it exists:

- If `--resume` was passed or the state file references the same interface:
  - `status: "connecting"` — restart Phase 1
  - `status: "enumerating"` — restart Phase 2
  - `status: "awaiting-scope"` — present the enumerated surface and ask the user to select scope
  - `status: "probing"` — resume Phase 3, skipping already-probed items (check findings array)
  - `status: "synthesizing"` — restart Phase 4
  - `status: "outputting"` — restart Phase 5 (output generation)
  - `status: "complete"` — inform the user the probe is done, ask if they want to re-probe
  - `status: "blocked"` — present the `blocked_reason` and ask the user how to proceed
- If the state file references a different interface, warn the user before proceeding.

If no state file exists, create one now (see State File format below).

---

## Phase 1: Connect & Preflight

Establish a connection to the interface and verify that you have read-only access. This phase is a safety gate — if anything looks wrong, stop.

### Connect

Use the provided connection info to establish a connection. If the connection fails, stop immediately and present the error. Do not retry — the user provides credentials, so the user fixes connectivity.

### Safety Checks

Run the interface-specific checks below, then assess the overall safety enforcement level.

#### Database

1. Attempt to open a read-only transaction (`BEGIN READ ONLY` or equivalent for the database engine).
2. If the database does not support read-only transactions, query the privilege system:
   - PostgreSQL: `SELECT * FROM information_schema.role_table_grants WHERE grantee = current_user AND privilege_type IN ('INSERT', 'UPDATE', 'DELETE', 'TRUNCATE')`
   - MySQL: `SHOW GRANTS FOR CURRENT_USER()`
   - SQLite: Check if the database file is opened read-only, or if the connection string specifies `mode=ro`
   - Other engines (SQL Server, Oracle, etc.): Attempt `BEGIN TRANSACTION READ ONLY` or the engine's equivalent. If the engine does not support read-only transactions and you cannot query its privilege system, warn the user that read-only access could not be verified and ask for confirmation before proceeding.
3. Determine which enforcement mechanisms are active (see Safety Enforcement Level below).
4. If the enforcement level is WEAK, **STOP** and present findings with a recommendation to upgrade (see below).

#### REST API

1. Probe will only issue GET and HEAD requests. This is non-negotiable.
2. If an OpenAPI/Swagger spec is available at the base URL (check `/openapi.json`, `/swagger.json`, `/api-docs`, `/.well-known/openapi`), fetch and parse it. Classify all endpoints by HTTP method. Record non-GET endpoints in the surface map but do not call them.
3. If no spec is available, discovery is limited to what can be found via GET requests (root endpoint, index pages, HATEOAS links, common API patterns).
4. Check for authentication requirements. If the first request returns 401/403, stop and ask the user for credentials.

#### MCP Server

1. Call `tools/list` to enumerate all available tools.
2. Classify each tool by reading its `description` and `inputSchema`:
   - **Safe**: Description clearly indicates read-only behavior (get, list, read, fetch, search, query, describe, show, count, check, status, info, inspect).
   - **Unsafe**: Description contains mutation language (create, delete, remove, modify, update, write, send, execute, trigger, post, put, patch, set, add, clear, reset, drop, insert, move, rename).
   - **Ambiguous**: Cannot determine from description and schema alone.
3. **Exclude ambiguous tools by default.** Do not call them unless the user explicitly overrides.
4. Present the classification to the user:

```
## MCP Tool Classification

### Safe (will probe)
- tool_name_1: "description..."
- tool_name_2: "description..."

### Excluded — Potentially Unsafe
- tool_name_3: "description..." [reason: contains "execute"]
- tool_name_4: "description..." [reason: ambiguous — no clear read/write signal]

### Excluded — Unsafe
- tool_name_5: "description..." [reason: contains "delete"]

To override, list the tool names you want to include:
> allow tool_name_3, tool_name_4
```

Wait for user confirmation before proceeding. Record the final classification in the state file.

### Safety Enforcement Level

After running interface-specific checks, classify the enforcement level. This tells the user — and you — how much the safety guarantee depends on probe's own discipline vs. infrastructure that cannot be bypassed.

#### Database

Detect which of the following is true, in order from strongest to weakest:

| Level | Condition | What it means |
|-------|-----------|---------------|
| **STRONG** | Connected to a read replica, or database is opened in read-only mode (e.g., SQLite `mode=ro`) | Mutations are rejected by the infrastructure regardless of what queries are issued. The database engine enforces safety. |
| **STRONG** | Connected user has only SELECT/read grants — no INSERT, UPDATE, DELETE, DROP, TRUNCATE, ALTER | Mutations are rejected by the privilege system. The database engine enforces safety. |
| **MODERATE** | Running inside a read-only transaction (`BEGIN READ ONLY` succeeded) but the user has mutation privileges | Mutations are rejected within the transaction, but queries issued outside the transaction (e.g., if the transaction is accidentally closed) would not be blocked. |
| **WEAK** | None of the above — relying solely on probe's query discipline | No infrastructure enforcement. Safety depends entirely on the LLM issuing only SELECT queries. This is the least reliable guarantee. |

**How to detect STRONG (read replica):**
- PostgreSQL: `SELECT pg_is_in_recovery()` — returns `true` on replicas
- MySQL: `SHOW REPLICA STATUS` (or `SHOW SLAVE STATUS` on pre-8.0.22) or `SELECT @@read_only` — `@@read_only = 1` on replicas
- SQL Server: `SELECT DATABASEPROPERTYEX(DB_NAME(), 'Updateability')` — returns `READ_ONLY` on replicas
- MongoDB: Check `rs.status()` for `SECONDARY` role
- For other engines: Check connection string for read-replica indicators, or query the engine's replication status

#### REST API

| Level | Condition |
|-------|-----------|
| **STRONG** | Requests are routed through an HTTP proxy that rejects non-GET/HEAD methods |
| **MODERATE** | API key or token has read-only scope (verifiable from the API's permission/scope endpoint if available) |
| **WEAK** | No verification possible — relying on probe's method discipline only |

REST API enforcement is typically WEAK unless the user confirms they have set up a proxy or read-only API credentials. Ask if either is in place.

#### MCP Server

| Level | Condition |
|-------|-----------|
| **MODERATE** | All tools classified as safe, and user has reviewed and approved the classification |
| **WEAK** | Ambiguous tools were overridden to be included, or tool descriptions are missing/vague |

MCP has no infrastructure-level read-only enforcement. The best available guarantee is the classification + user approval flow.

### Present Enforcement Level

After determining the level, present it clearly:

```
## Safety Enforcement Level: [STRONG / MODERATE / WEAK]

[For STRONG:]
Connected to a read replica / read-only user. Mutations are rejected by the database engine regardless of what queries probe issues. This is the strongest safety guarantee.

[For MODERATE:]
Running inside a read-only transaction. Mutations are rejected within the transaction. This is a good guarantee but not airtight — it depends on all queries being issued within the transaction.

[For WEAK:]
No infrastructure-level enforcement detected. Safety depends entirely on probe issuing only read-only queries.

Recommendation: Before proceeding, consider one of:
- Connect to a read replica instead of the primary
- Connect as a user with only SELECT/read grants
- Set up a read-only transaction (probe will attempt this automatically)

You may proceed anyway, but the safety guarantee is weaker. Confirm to continue.
```

For **STRONG**: proceed after informing the user.

For **MODERATE**: proceed after informing the user, but note the limitations.

For **WEAK**: **HARD STOP.** Do not proceed until the user has explicitly addressed the safety gap. Present the following:

```
## Safety Enforcement: WEAK — Action Required

No infrastructure-level enforcement detected. Probe cannot guarantee that all operations are safe — it can only promise to try. If something goes wrong, the damage is real.

Before proceeding, you must either upgrade the enforcement level or accept responsibility:

Option 1 (recommended): Upgrade to STRONG
- Connect to a read replica instead of the primary
- Or connect as a database user with only SELECT/read grants
- Then re-run /probe with the new connection info

Option 2: Upgrade to MODERATE
- Confirm that you have set up a read-only transaction context
- Or confirm that your API credentials are scoped to read-only access

Option 3: Accept WEAK enforcement
- You must explicitly state: "I accept that there is no infrastructure-level safety enforcement. I take responsibility for any unintended side effects."
- Probe will proceed using query discipline only. This is the least reliable guarantee.

Which option?
```

Do not accept vague confirmations like "sure" or "go ahead." The user must either provide upgraded connection info (Options 1/2) or state the explicit acceptance phrase (Option 3). This is not a formality — it establishes that the user, not the agent, owns the risk when infrastructure enforcement is absent.

Record the user's exact response in the state file's `preflight.user_acknowledgment` field, along with a timestamp.

### Phase 1 Exit

- Connection verified
- Safety enforcement level determined and presented
- Read-only access confirmed at the detected enforcement level (or user override recorded for WEAK)
- Tool/endpoint/table classification complete (for MCP)
- Update state: `status: "enumerating"`, record `preflight.enforcement_level`

---

## Phase 2: Enumerate Surface

Perform a cheap discovery pass. List everything without deep inspection. The goal is to give the user a map of what exists so they can choose what to explore in depth.

### Database

Query the schema metadata:

1. List all tables and views with row counts (if available without full table scan — use statistics tables where possible)
2. For each table/view: column names, types, nullable, primary keys, foreign keys
3. List indexes, constraints, and triggers (names only — not definitions)
4. If the database has schemas/namespaces, organize by schema

Present as a structured summary:

```
## Database Surface: <name>

### Tables (N)
- orders (estimated ~50k rows, 12 columns, 3 indexes, 2 FK relationships)
- users (estimated ~10k rows, 8 columns, 2 indexes)
- ...

### Views (N)
- active_orders (references: orders, users)
- ...

### Schemas
- public: N tables, M views
- analytics: N tables, M views

Which tables/views do you want to probe in depth? You can specify:
- Individual names: orders, users
- Schemas: public.*
- All: *
```

### REST API

Enumerate available endpoints:

1. If an OpenAPI spec was found in Phase 1, extract all GET endpoints with their paths, parameters, and response schemas
2. If no spec: start from the base URL, follow links, try common patterns (`/api/v1`, `/health`, `/status`, etc.)
3. Note any pagination patterns, filtering parameters, or query options visible in the spec or responses

Present as a structured summary:

```
## REST API Surface: <name>

### Endpoints (N GET endpoints discovered)
- GET /api/v1/users — List users (params: page, limit, sort)
- GET /api/v1/users/{id} — Get user by ID
- GET /api/v1/orders — List orders (params: page, limit, status, date_from, date_to)
- ...

### Non-GET Endpoints (recorded, will not call)
- POST /api/v1/users — Create user
- DELETE /api/v1/users/{id} — Delete user
- ...

### Authentication
- Type: Bearer token / API key / None detected

Which endpoints do you want to probe in depth?
```

### MCP Server

Enumerate safe tools:

1. For each tool classified as safe: name, description, input schema, output schema (if available)
2. Group by functional area if patterns are apparent (e.g., "user-related tools", "data query tools")

Present as a structured summary:

```
## MCP Server Surface: <name>

### Safe Tools (N)
- get_user: "Retrieve user by ID" — input: { id: string } → output: User object
- list_orders: "List all orders with filtering" — input: { status?: string, limit?: number }
- ...

### Excluded Tools (M)
[listed in Phase 1]

Which tools do you want to probe in depth?
```

### Cost Warning

If the surface contains more than 100 items (tables, endpoints, or tools), append this warning:

```
WARNING: This interface has [N] items. Deep probing all of them will be expensive.
Consider selecting a focused subset. You can always re-run with additional scope later.
```

### Scope Selection

After presenting the surface, update state: `status: "awaiting-scope"`. This ensures that if the conversation is interrupted while waiting for user input, the resume handler will re-present the surface.

Once the user selects their scope, proceed.

### Phase 2 Exit

- Full surface enumerated and presented to user
- User has selected scope for deep probing
- Update state: `status: "probing"`, record approved scope with timestamp

---

## Phase 3: Deep Probe

Systematically explore each item in the approved scope. Record every finding with evidence — the actual response data, not a summary.

### Database

For each approved table or view:

1. **Schema detail**: Full column definitions including types, defaults, constraints, comments
2. **Row count**: Use estimated counts from statistics tables (e.g., `pg_stat_user_tables`, `information_schema.tables`, `SHOW TABLE STATUS`) when available. Only run `SELECT COUNT(*)` on tables with estimated row counts under 100k. For larger tables, record the estimate and note it as approximate. A `COUNT(*)` on a multi-million-row table can lock the table, saturate I/O, and impact production — this is not acceptable for a read-only probe.
3. **Sample data**: `SELECT * FROM <table> LIMIT 5` — observe actual data shapes, null patterns, value ranges
4. **Null frequency**: Combine null checks into a single query per table, not one query per column. Use portable syntax: `SUM(CASE WHEN <col> IS NULL THEN 1 ELSE 0 END)` — the `FILTER` clause is PostgreSQL-specific and will fail on other engines. For tables with estimated row counts over 100k, sample instead of scanning: `SELECT ... FROM <table> TABLESAMPLE SYSTEM(1)` (PostgreSQL) or `TABLESAMPLE` (SQL Server). Note that frequencies from samples are approximate. Most other engines (MySQL, SQLite, MongoDB) have no `TABLESAMPLE` equivalent — on those engines, skip null frequency analysis on large tables and record this as a limitation.
5. **Distinct values**: For columns with low cardinality (< 50 distinct values based on table statistics or a `SELECT COUNT(DISTINCT <col>)` check), retrieve the full set of distinct values. These are often enums or status fields.
6. **Foreign key relationships**: Follow FK references. Note which relationships are enforced vs. advisory.
7. **Indexes**: List indexes with their columns and types (btree, hash, gin, etc.). Note missing indexes on FK columns.
8. **Empty tables**: If a table has 0 rows, still probe the schema fully. Record "empty table" as a finding — it may indicate an unused feature, a staging table, or incomplete migration.
9. **Unusual types**: Note columns with JSON, array, geometry, or other complex types. Sample values to understand actual structure.

**All queries MUST be wrapped in a read-only transaction where possible.**

#### Database Engine Safety

The following are known performance hazards for common database engines. Obey these constraints in addition to the read-only invariant.

**Relational databases (PostgreSQL, MySQL, SQL Server, Oracle, SQLite):**
- Never run `SELECT COUNT(*)` on tables with estimated row counts over 100k. Use statistics tables.
- Never run `SELECT *` without `LIMIT`. Always cap at 5 rows for samples.
- Avoid full table scans for analytical queries (null frequency, distinct values) on large tables. Use sampling or skip the check.
- On MySQL, avoid `SHOW TABLE STATUS` on InnoDB tables with hundreds of tables — it can be slow. Prefer `information_schema.tables`.

**Graph databases (Neo4j, Neptune, ArangoDB):**
- Never run unbounded traversals. Always set an explicit depth limit on path queries (e.g., `MATCH path = (a)-[*1..3]-(b)` in Cypher, not `[*]`).
- Count queries on large graphs (`MATCH (n) RETURN count(n)`) can be expensive. Use `CALL db.stats.retrieve()` or equivalent metadata endpoints for counts.
- Label/relationship scans without property filters can be as expensive as full table scans. Always filter when possible.
- On Neptune, SPARQL `SELECT (COUNT(*) ...)` over the entire graph is a full scan. Use the Neptune statistics API instead.

**Document databases (MongoDB, CouchDB, DynamoDB):**
- On MongoDB, `db.collection.count()` without a filter is fast (uses metadata), but `db.collection.countDocuments({})` with an empty filter scans the collection. Use `estimatedDocumentCount()` for large collections.
- Avoid `find({})` without `.limit()` — always cap results.
- On MongoDB, avoid running `$sample` on unsharded collections larger than 100MB without checking — it can be slow.
- On DynamoDB, `Scan` operations consume read capacity units proportional to the table size, not the result size. Use `DescribeTable` for metadata and count estimates. Only `Query` with a partition key, never `Scan`, for data sampling.
- On CouchDB, avoid view queries that trigger index rebuilds.

**For any database engine not listed above:** Before issuing any exploratory queries, perform a web search for "[engine name] expensive read queries to avoid" and "[engine name] read-only query performance hazards." Incorporate what you learn into your probing strategy. Record any engine-specific constraints you discover in the state file's `preflight.warnings` array. The goal is to never surprise a DBA with unexpected load from a read-only probe.

### REST API

For each approved endpoint:

1. **Minimal call**: Call with only required parameters. Record response status, headers, body shape, and timing.
2. **Full call**: Call with all optional parameters populated with reasonable values. Compare response to minimal call.
3. **Pagination**: If the endpoint supports pagination, test: first page, second page, last page (if discoverable), page beyond the end. Check for off-by-one, empty last pages, and consistency between total count and actual pages.
4. **Filtering**: If parameters suggest filtering (date ranges, status values, search), test several values. Verify that filters actually filter (compare filtered vs. unfiltered counts).
5. **Invalid inputs**: Test with wrong types, missing required params, out-of-range values. Record the error format — is it consistent? Structured? Useful?
6. **Empty results**: Test with filters that should return nothing. Check whether the response is an empty array, null, a 404, or something else.
7. **Response consistency**: Call the same endpoint twice. Verify the response schema is stable (same fields, same types). Note any non-deterministic fields.
8. **Rate limit headers**: Record `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`, or any equivalent headers on every response.

**If any response returns 429 or rate limit headers indicating exhaustion**: STOP immediately. Do not attempt to wait and retry. Save state and alert the user (see Rate Limit Handling below).

### MCP Server

For each approved tool:

1. **Minimal call**: Call with only required parameters. Record the response structure and content.
2. **Optional parameters**: Call with optional parameters populated. Compare response to minimal call.
3. **Edge inputs**: Test with edge-case values. Limit aggressive testing (long strings, boundary values) to tools the user has explicitly approved for deep probing — even a "safe" tool can behave unpredictably with adversarial input.
   - Empty string where a string is expected
   - Zero where a number is expected
   - Moderately long string (200 chars) if a string parameter has no stated limit — do not send excessively large payloads that could cause resource exhaustion or error log noise
   - Null/undefined for optional parameters
4. **Missing required params**: Call without required parameters. Record the error — is it a clear validation error or an opaque failure?
5. **Invalid types**: Pass a string where a number is expected, an array where a string is expected. Record validation behavior.
6. **Response consistency**: Call the same tool twice with the same input. Verify the response is consistent.
7. **Error format**: Across all error cases, is the error format consistent? Structured? Does it include error codes?

### Progress Checkpoints

If the approved scope contains more than 20 items, write the state file after every 10 items probed. This serves two purposes: recovery on interruption, and a visible progress signal. After each checkpoint, print a brief status line:

```
Probed 10 of 47 items. [N] findings so far. State saved.
```

### Recording Findings

Every observation becomes a finding in the state file. Use these categories:

- **capability**: Something the interface can do, confirmed by observed behavior
- **gap**: Something the interface cannot do, or functionality that is missing or incomplete
- **failure**: Something that broke — errors, unexpected behavior, inconsistencies
- **edge-case**: Boundary behavior observed — empty results, null handling, type coercion, pagination limits

Each finding must include:
- `surface_id`: Which item this relates to (e.g., SURF-1)
- `category`: One of the four above
- `description`: What was observed
- `evidence`: The actual response data, query, or error message

### Rate Limit Handling

On detection of any rate limit signal (HTTP 429, `Retry-After` header, `X-RateLimit-Remaining: 0`, repeated 503s, or MCP tool errors suggesting throttling):

1. Stop all probing immediately
2. Record what was completed and what remains
3. Update state: `status: "blocked"`, `blocked_reason: "rate_limit"`, `rate_limit_detected: true`
4. Present to the user:

```
## Probe Paused — Rate Limit Detected

[What was detected: specific header or status code]
[What was completed: N of M items probed]
[What remains: list of unprobed items]

State saved to .probe-state-<name>.json.

To resume when the rate limit resets:
  /probe --resume
```

Do not attempt to wait and retry. Do not attempt to slow down and continue. Stop and let the user decide.

### Phase 3 Exit

- All approved items probed, or blocked by rate limit / error
- All findings recorded with evidence
- Update state: `status: "synthesizing"` (or `"blocked"` if stopped early)

---

## Phase 4: Synthesize

Organize raw findings into a coherent capability profile. This is analysis, not just reorganization — identify patterns, relationships, and workflows that aren't obvious from individual findings.

### Capabilities

Group findings by functional area, not by table/endpoint/tool. What can this interface *do*?

- For a database: "Supports order lifecycle tracking (orders, order_items, order_status_history tables form a complete audit trail)"
- For a REST API: "Provides full CRUD for users and read-only access to analytics (no write endpoints for analytics data)"
- For an MCP server: "Offers document search and retrieval but no document creation or modification"

### Gaps

What is the interface missing? What would you expect to exist but doesn't?

- Missing functionality: "No endpoint for bulk operations — each item must be fetched individually"
- Incomplete features: "The /search endpoint accepts a `date_range` parameter but ignores it (filtered and unfiltered results are identical)"
- Undocumented limitations: "The `limit` parameter caps at 100 even when higher values are requested — no error, just silently truncated"

### Failures

What broke during probing?

- Errors: "GET /api/v1/reports returns 500 with no body when `format=csv` is specified"
- Inconsistencies: "The `total_count` field in paginated responses doesn't match the actual number of items across all pages"
- Unexpected behavior: "Querying the `events` table with `WHERE timestamp > '2024-01-01'` returns results from 2023"

### Edge Cases

Boundary behaviors observed:

- "Empty collections return `[]` for /users but `null` for /orders"
- "The `description` column in the `products` table contains NULLs for 40% of rows despite being marked NOT NULL in the schema"
- "Requesting page 0 returns the same results as page 1"

### Workflow Recommendations

Identify compound operations that work well together, common patterns, and pitfalls:

- "To get a complete order with items and customer info, join orders → order_items → products and orders → users. The FK relationships support this directly."
- "The search endpoint returns IDs only. To get full objects, follow up with individual GET calls. There is no batch endpoint — expect N+1 for search results."
- "Rate limits are per-endpoint, not global. Parallelizing across different endpoints is safe."

### User Review

Present the synthesized profile to the user before writing output artifacts. The synthesis makes interpretive judgments — classifying gaps, identifying failures, recommending workflows — that the user must have the opportunity to challenge or correct.

```
## Synthesis Review

Here is the capability profile I've assembled from the probe findings:

### Capabilities
[list capabilities with evidence]

### Gaps
[list gaps with evidence]

### Failures
[list failures with evidence]

### Edge Cases
[list edge cases]

### Workflow Recommendations
[list recommendations]

Please review before I write the final artifacts. You may:
- Correct any misclassifications (e.g., "that's not a gap, it's intentional")
- Add context I couldn't observe (e.g., "that table is empty because it's for a feature launching next month")
- Remove or adjust workflow recommendations
- Say "looks good" to proceed to output
```

Incorporate any user feedback before proceeding. If the user reclassifies findings, update the state file's findings array accordingly.

### Phase 4 Exit

- All findings organized into capabilities, gaps, failures, edge cases, and workflows
- User has reviewed and approved the synthesis
- Update state: `status: "outputting"`

---

## Phase 5: Output

Generate the final artifacts.

### Create Output Directory

Create `probes/probe-<interface-name>/` if it doesn't exist.

### Write `profile.md`

The main artifact. Structure depends on mode.

**Both modes use this frontmatter:**

```yaml
---
interface: <name>
type: database | rest-api | mcp-server
probed_at: <ISO timestamp>
mode: human | machine
scope: <list of items that were deeply probed>
surface_total: <total items discovered in enumeration>
connection: <sanitized connection info — no passwords, keys, or tokens>
probe_version: "1.0"
---
```

**Human mode:** Narrative prose with inline examples. Each section reads like a briefing document. Sample data is formatted as readable tables or code blocks. Workflow recommendations include concrete examples.

**Machine mode:** Each section uses structured YAML blocks. Capabilities, gaps, failures, and edge cases are typed entries with consistent fields:

```yaml
capabilities:
  - id: CAP-1
    area: "order management"
    description: "Full order lifecycle tracking with audit trail"
    evidence:
      - surface_id: SURF-1
        observation: "orders table with status, created_at, updated_at columns"
      - surface_id: SURF-3
        observation: "order_status_history table with FK to orders"
    related: [CAP-2, CAP-3]

gaps:
  - id: GAP-1
    area: "bulk operations"
    description: "No mechanism for retrieving multiple items by ID in a single call"
    evidence:
      - surface_id: SURF-5
        observation: "GET /users/{id} accepts single ID only, no batch endpoint exists"
    impact: "N+1 queries required for any list-then-detail workflow"
```

This format is designed to be consumed by `/spec` (gaps become requirements to address) or `mcp-topo` (capabilities map to tool surfaces).

### Write `raw-findings.json`

Structured data from the deep probe. One entry per finding, with full evidence. This file is the source of truth for delta comparisons.

```json
{
  "interface_name": "<name>",
  "interface_type": "<type>",
  "probed_at": "<ISO>",
  "findings": [
    {
      "id": "F-1",
      "surface_id": "SURF-1",
      "category": "capability",
      "description": "...",
      "evidence": "...",
      "probed_at": "<ISO>"
    }
  ]
}
```

### Write `surface-map.md`

The full enumeration from Phase 2, including items that were not deeply probed. This serves as a reference for what exists and enables future runs to expand scope.

### Delta Mode

If `--diff probes/probe-<name>/` was provided:

1. Read `raw-findings.json` from the previous probe output directory
2. Match findings between runs using the composite key: `surface_id` (matched by surface item **name**, not the auto-generated SURF-N ID, since IDs may differ between runs) + `category`. Two findings match if they reference the same surface item name and the same category. When multiple findings share the same key, compare descriptions for semantic similarity.
3. Classify each finding:
   - **Added**: Findings in the current run that have no match in the previous run (new capabilities, new failures)
   - **Removed**: Findings in the previous run that have no match in the current run (resolved failures, removed capabilities)
   - **Changed**: Findings that match by key but have different evidence or description (a former failure now works, a capability now behaves differently)
4. Write `delta.md` to the output directory:

```markdown
---
compared_to: <path to previous probe>
previous_probe_date: <ISO>
current_probe_date: <ISO>
---

# Delta Report: <interface-name>

## Added
[New capabilities, gaps, failures, or edge cases not present in the previous probe]

## Removed
[Items from the previous probe no longer observed]

## Changed
[Items present in both probes but with different behavior]

## Unchanged
[Summary count: N items unchanged out of M total]
```

### Phase 5 Exit

- All artifacts written
- Update state: `status: "complete"`
- Print terminal summary:

```
## Probe Complete

Interface: <name> (<type>)
Surface: <N> items discovered, <M> deeply probed
Findings: <N> capabilities, <N> gaps, <N> failures, <N> edge cases

Output: probes/probe-<interface-name>/
  - profile.md (capability profile, <mode> mode)
  - raw-findings.json (structured data)
  - surface-map.md (full surface enumeration)
  [- delta.md (comparison with previous probe)]
```

---

## State File

Maintain `.probe-state-<name>.json` in the project root throughout the run. Write it after every phase transition — this is what allows recovery after context compaction or interruption.

```json
{
  "interface_name": "<name>",
  "interface_type": "database | rest-api | mcp-server",
  "started": "<ISO timestamp>",
  "last_updated": "<ISO timestamp>",
  "status": "connecting | enumerating | awaiting-scope | probing | synthesizing | outputting | complete | blocked",
  "connection": {
    "sanitized_info": "<connection info without secrets>"
  },
  "preflight": {
    "passed": true,
    "enforcement_level": "strong | moderate | weak",
    "enforcement_detail": "<how the level was determined — e.g., 'pg_is_in_recovery() returned true'>",
    "read_only_verified": true,
    "unsafe_operations_detected": [],
    "warnings": [],
    "user_acknowledgment": "<exact user response if enforcement is WEAK, null otherwise>",
    "user_acknowledgment_at": "<ISO timestamp or null>"
  },
  "surface": {
    "total_items": 0,
    "items": [
      {
        "id": "SURF-1",
        "name": "<table/endpoint/tool name>",
        "type": "table | view | endpoint | tool",
        "metadata": {},
        "approved_for_deep_probe": false
      }
    ]
  },
  "scope": {
    "approved": ["SURF-1", "SURF-3"],
    "excluded": ["SURF-2"],
    "approved_at": "<ISO timestamp>"
  },
  "findings": [
    {
      "id": "F-1",
      "surface_id": "SURF-1",
      "category": "capability | gap | failure | edge-case",
      "description": "<what was observed>",
      "evidence": "<actual response data>",
      "phase": "deep-probe"
    }
  ],
  "output": {
    "mode": "human | machine",
    "path": "probes/probe-<name>/",
    "diff_against": null
  },
  "phase_history": [
    {
      "phase": "connect",
      "started_at": "<ISO timestamp>",
      "completed_at": "<ISO timestamp>",
      "outcome": "passed | failed | skipped"
    },
    {
      "phase": "enumerate",
      "started_at": "<ISO timestamp>",
      "completed_at": "<ISO timestamp>",
      "items_discovered": 47
    },
    {
      "phase": "deep-probe",
      "started_at": "<ISO timestamp>",
      "completed_at": "<ISO timestamp or null if interrupted>",
      "items_probed": 15,
      "items_total": 47,
      "findings_count": 32,
      "interrupted_reason": null
    }
  ],
  "rate_limit_detected": false,
  "blocked_reason": null
}
```

Valid `status` values: `"connecting"`, `"enumerating"`, `"awaiting-scope"`, `"probing"`, `"synthesizing"`, `"outputting"`, `"complete"`, `"blocked"`.

---

## Notes

- Never issue mutating operations. This is the one inviolable rule. SELECT, GET, HEAD, and read-only tool calls only.
- The user provides credentials. Do not store secrets in state files or output artifacts. Sanitize connection info in all persisted data.
- Always write state after every phase transition. Do not rely on conversational context alone.
- If the surface is very large, warn about token cost before deep probing. Let the user scope aggressively.
- Findings are observations, not judgments. Report what is, not what should be. "This column is 40% NULL" is a finding. "This column should not allow NULLs" is an opinion — save it for workflow recommendations if it matters.
- If a connection fails mid-probe, do not retry. Save state, report what happened, and let the user investigate.
- If a rate limit is detected, stop immediately. Do not wait, do not slow down, do not retry. Save state and alert the user.
- On resume, check which surface items already have findings and skip them. Do not re-probe completed items unless the user explicitly requests it.
