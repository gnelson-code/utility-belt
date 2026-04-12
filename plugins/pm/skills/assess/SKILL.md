---
name: assess
description: |
  Adversarial risk assessment for drug pipeline critical path.
  Interviews for portfolio context, drafts a structured risk register, then runs a critic sub-agent through iterative stress-tests (Opus → Sonnet) until no Critical/Major findings remain or iteration 4 hits.
  User always wins disagreements; critic objections are logged as dissents for audit trail.
user-invocable: true
argument-hint: "[input-file-path ...] [--context <name-or-path>] [--list]"
---

You are the orchestrator of a risk assessment session. Your job is to produce a structured risk register for a drug pipeline portfolio that has been adversarially stress-tested by an independent critic. You handle the interview, the drafting, and the triage. The critic handles the challenge.

The user is the domain expert. They always win disagreements. Your job is to make them *think*, not to overrule them.

### Severity Levels

Used by both you and the critic. These match the mob taxonomy for consistency.

- **Critical**: a flaw that makes the assessment unsafe to rely on — a missed high-impact risk, a clearly wrong severity score, a factually mistaken critical-path linkage, or a context gap that undermines the analysis.
- **Major**: materially degrades the quality or completeness of the assessment — an unjustified severity score, an ineffective mitigation, an absent risk category, a dropped risk that should have been kept.
- **Minor**: reduces quality but does not compromise the assessment — phrasing ambiguity, missing detail, a mitigation that could be stronger.
- **Suggestion**: out of scope for this iteration but worth noting.

### Artifact Locations

All artifacts go under `notes/risk/`:

- Portfolio context: `notes/risk/portfolio-{slug}-{YYYY-MM-DD}.md`
- Risk register: `notes/risk/register-{slug}-{YYYY-MM-DD}.md`
- Executive summary: `notes/risk/exec-summary-{slug}-{YYYY-MM-DD}.md`

---

## Setup

### Parse Arguments

- **Positional arguments** (file paths) → input files to analyze before the interview. Accepts the same formats as `/pm:timeline`: unstructured docs (markdown, plain text, PDF) or structured data (Excel `.xlsx`/`.xls`, YAML `.yaml`/`.yml`, JSON `.json`). Multiple files can be provided. These are the primary data seed — the interview fills gaps the files leave open.
- `--list` → print every `notes/risk/portfolio-*.md` file with its slug, date, and the first non-frontmatter heading, then **stop**. Do not enter the interview or analysis flow.
- `--context <name-or-path>` → load an existing portfolio context file. If the argument is a bare name (no path separator), look for `notes/risk/portfolio-{name}*.md` and use the most recent match. If the file exists, enter **Incremental Update** mode. If it does not exist, tell the user and offer to start a fresh interview.
- No arguments and no files → enter **Fresh Interview** mode with no data seed.

---

## Fresh Interview

The goal is to build a portfolio context file that is specific enough for the critic to attack later. Analyze input files first, then interview for what's missing.

### Step 1: Analyze input files

If the user provided input files (positional arguments) or pasted content into the prompt, this is your primary data seed. Analyze it **before** asking any questions.

**Structured files (Excel, YAML, JSON):**

- If the file matches the `/pm:timeline` canonical schema (has `workstreams` and `tasks` keys in YAML/JSON, or columns matching the timeline column synonyms in Excel), extract: program name(s) from the title or workstream names, task names and dates (which reveal stage and timeline), status values (which reveal what's at risk, blocked, or delayed), and workstream structure (which reveals functional areas in play).
- If the file is a prior risk register or assessment, extract: programs, risk categories already identified, severity scores, mitigations, and critical-path descriptions.
- If the file is some other structured format, extract whatever maps to the required fields below.

**Unstructured files (markdown, plain text, PDF):**

- Read the full document. Extract anything matching the required or soft fields below: program names, regulatory pathways, development stages, timeline references, dependency descriptions, risk language, milestone descriptions.
- Pay attention to language that implies critical path without naming it explicitly — phrases like "rate-limiting," "must complete before," "on the critical path," "blocking," "long-lead," "pacing item."

**Multiple files:** Analyze all files. Cross-reference — a timeline file gives you dates and structure; a strategy doc gives you rationale and context. Information from one file may fill gaps left by another.

After analysis, you should have a partial picture of the portfolio. Proceed to Step 2 with whatever you extracted. The interview (Steps 3–4) exists only to fill what the files left open.

### Step 2: Required fields

The following fields are **mandatory**. The skill must not proceed to register drafting without them.

| Field | What it means | Why it's load-bearing |
|-------|--------------|----------------------|
| **Programs in scope** | Which drug programs are being assessed (at least one) | Scopes the entire analysis |
| **Approval pathway** | Per program: full vs. conditional/accelerated, which authority (FDA / EMA / PMDA / other) | Different pathways have different rate-limiters |
| **Current stage** | Per program: preclinical, Phase 1/2/3, NDA/BLA prep, under review, etc. | Stage determines which risk categories matter |
| **Critical path** | What activities and milestones are on the critical path to approval, AND **why** each is rate-limiting | This is the core of the assessment — without a "why", the critic has nothing to attack |
| **Dependencies and rate-limiters** | Upstream/downstream dependencies, vendor/CMO commitments, read-out timelines, external constraints | Risks without dependency context are speculation |

### Step 3: Summarize known vs. missing

Present a concrete table to the user:

```
## What I have so far

| Field                        | Status         | Detail                    |
|------------------------------|----------------|---------------------------|
| Programs in scope            | ✓ / missing    | {extracted value or "—"}  |
| Approval pathway             | ✓ / missing    | ...                       |
| Current stage                | ✓ / missing    | ...                       |
| Critical path                | ✓ / partial    | ...                       |
| Dependencies / rate-limiters | ✓ / missing    | ...                       |
```

### Step 4: Ask targeted questions

For each missing required field, ask a specific question. Batch questions where possible — do not ask them one at a time. Do **not** re-ask for fields already extracted from the user's input.

If critical path is listed but lacks a **why** for any item, ask: "You listed {item} as critical path — what makes it rate-limiting?"

### Step 5: Optional depth

After required fields are complete, ask once: "Want me to also capture team ownership, prior incidents, regulatory correspondence history, or competitive landscape? These aren't required but sharpen the analysis." Accept the user's choice; do not push.

### Step 6: File naming

Present the default filename and let the user accept or customize:

```
I'll save this as `portfolio-{default-slug}-{YYYY-MM-DD}.md`.
Options:
  1. Use this name
  2. Enter a custom name
```

The default slug is derived from the program name(s), lowercased and hyphenated (e.g., `portfolio-program-alpha-2026-04-11.md`). If multiple programs, use the most prominent or combine (e.g., `portfolio-alpha-beta-2026-04-11.md`).

### Step 7: Draft the portfolio context file

Write the file to `notes/risk/{chosen-filename}.md` using this template:

```markdown
# Portfolio Context: {title}

**Slug:** {slug}
**Created:** {YYYY-MM-DD}
**Last updated:** {YYYY-MM-DD}

## Programs in scope

- **{Program name}** — {pathway} ({authority}), currently {stage}
- ...

## Critical path

{Per program or aggregate: a bulleted description of what's on critical path and why each item is rate-limiting. Every item must have a "why".}

## Dependencies and rate-limiters

- ...

## Optional context

{Only populated if user opted in. Otherwise this section reads: "Not captured in this assessment."}

## Revision history

- {YYYY-MM-DD}: Initial draft
```

### Step 8: Interview critique

After drafting the context file, spawn the `risk-critic` agent as a **fresh sub-agent** with `model: sonnet`.

**Prompt:**

```
MODE: Interview-sufficiency

You are reviewing a portfolio context file for an upcoming risk assessment.
Your ONLY job at this stage is to decide whether this context is SUFFICIENT
to perform a rigorous risk analysis, or whether there are load-bearing
unknowns that would make the resulting analysis unreliable.

Do NOT generate risks. Do NOT critique severity or likelihood. Only evaluate
whether the required fields are populated with enough specificity that the
register critique later has something concrete to attack.

Portfolio context file:

[paste full file contents]
```

**After the critic returns:**

- **Critical findings:** Present each finding to the user. Ask the targeted question the critic suggested. Update the context file with the user's response. Max **2 interview-critique loops total**. If still Critical after loop 2, add a `## Context gaps` section to the context file listing the unresolved gaps, then proceed anyway.
- **Major findings only:** Note them but proceed to register drafting.
- **No findings:** Proceed.

Confirm the final context file path to the user before proceeding.

---

## Draft Risk Register

Read the finalized portfolio context file. For each program in scope, systematically consider risks across these categories:

- Regulatory
- Clinical data
- CMC / manufacturing
- Commercial
- Operational
- IP
- Financial
- Competitive

Not every category will produce a risk for every program. Do not force risks into existence — if a category is genuinely not applicable given the program's stage and pathway, skip it. But do not skip a category simply because it is harder to assess.

For each risk you identify, evaluate:

1. **Critical path linkage** — does this risk threaten a specific item on the critical path described in the portfolio context? If not, it may still be worth noting but it is not a critical-path risk. Be honest about linkage.
2. **Likelihood** — Low / Medium / High. Justify in one sentence.
3. **Impact** — Low / Medium / High. Justify in one sentence.
4. **Severity** — derived from likelihood × impact. Use this matrix:

| | Low Impact | Medium Impact | High Impact |
|---|---|---|---|
| **Low Likelihood** | Low | Low | Medium |
| **Medium Likelihood** | Low | Medium | High |
| **High Likelihood** | Medium | High | Critical |

5. **Mitigations** — concrete actions that would reduce likelihood or impact. Each mitigation must be specific enough that someone could execute it. "Monitor closely" is not a mitigation.
6. **Residual risk** — severity after mitigations are implemented.

### Register format

Write to `notes/risk/register-{slug}-{YYYY-MM-DD}.md`:

```markdown
# Risk Register: {title}

**Portfolio context:** {path to context file}
**Created:** {YYYY-MM-DD}
**Iterations run:** 0
**Status:** draft

## Risks

### R-001: {short title}

- **Description:** {what the risk is, concretely}
- **Category:** {Regulatory | Clinical data | CMC | Commercial | Operational | IP | Financial | Competitive}
- **Critical path linkage:** {which critical-path item this threatens and how — or "None (secondary risk)"}
- **Likelihood:** {Low | Medium | High} — {one-sentence justification}
- **Impact:** {Low | Medium | High} — {one-sentence justification}
- **Severity:** {Low | Medium | High | Critical} — {one-sentence justification referencing likelihood × impact}
- **Rationale:** {the analyst's reasoning for these scores — this is what the critic will attack, so make it defensible}
- **Mitigations:**
  - {concrete mitigation 1}
  - {concrete mitigation 2}
- **Residual risk after mitigation:** {Low | Medium | High} — {one-sentence justification}
- **Dissents:**

### R-002: ...

---

## Considered and dropped

| Risk idea | Category | Why dropped |
|-----------|----------|-------------|
| {description} | {category} | {one sentence — must explain why it does not touch critical path or is too low-impact to track} |
```

After writing the draft register, proceed immediately to the Critic Loop.

---

## Critic Loop

The register is stress-tested through iterative critic passes. The loop runs a maximum of **4 iterations**. Each iteration: spawn critic → receive findings → triage with user → update register.

### Per-iteration flow

#### Step 1: Spawn the critic

Spawn `risk-critic` as a **fresh sub-agent** every iteration. Never reuse agents across iterations.

**Model selection** (use the Agent tool's `model` parameter):
- **Iteration 1:** `model: opus`
- **Iterations 2–4:** `model: sonnet`

**Prompt:**

```
MODE: Register critique
ITERATION: {N} of 4

Portfolio context:

[paste full portfolio context file]

Risk register (current draft):

[paste full register contents]

Review this register against the portfolio context. Use your standard attack
vectors: unjustified scores, weak critical-path linkages, ineffective
mitigations, overlooked risks in the "considered and dropped" section, and
entirely missing risk categories.

You MUST ask "why do you think this?" about at least one risk whose rationale
feels most like an assertion rather than an argument.

Use the severity classification in your instructions.
```

#### Step 2: Receive findings

Wait for the critic to return its structured findings list.

#### Step 3: Triage with the user

Present **only Critical and Major findings** to the user. Group by risk ID. For each finding, present:

```
### [{severity}] {risk_id}: {claim}

**Critic says:** {justification}
**Suggested action:** {suggested_action}

Your call:
  (A) Accept — I'll update the register
  (D) Dissent — log the objection, keep my scores
  (F) Defer — treat as Minor, include in final report
```

Wait for the user's response on each finding. Do not batch — disagreements need individual attention.

Minor and Suggestion findings: accumulate silently for the final report. Do not present them during the loop.

#### Step 4: Apply changes

For each **accepted** finding:
- Update the affected risk entry in the register (revise scores, add mitigations, promote from dropped, or create a new risk entry).
- Do not add commentary about the iteration — just update the risk cleanly.

For each **dissented** finding:
- Add an entry under `Dissents:` on the affected risk:
  ```
  - `{YYYY-MM-DD}` critic claim: "{the critic's objection verbatim}" | user rationale: "{the user's stated reason for holding firm}"
  ```

For each **deferred** finding:
- Log it as a Minor for the final report. No register change.

Increment the `Iterations run:` count in the register header.

#### Step 5: Loop decision

| Condition | Action |
|-----------|--------|
| Zero Critical/Major findings from the critic | Exit loop → proceed to Executive Summary |
| All Critical/Major findings were dissented or deferred (none accepted) | Exit loop → proceed to Executive Summary |
| At least one finding was accepted AND this is iteration 1–3 | Update the register file, begin next iteration |
| Iteration 4 just completed | Exit loop regardless → proceed to Executive Summary. Log any unresolved accepted findings as a note in the exec summary. |

---

## Executive Summary

Produced **only** after the critic loop exits. This is the last artifact generated.

### Step 1: Finalize the register

Update the register file:
- Set `Status:` to `finalized`
- Ensure `Iterations run:` is accurate

### Step 2: Generate the executive summary

Write to `notes/risk/exec-summary-{slug}-{YYYY-MM-DD}.md`:

```markdown
# Risk Assessment Executive Summary: {title}

**Assessment date:** {YYYY-MM-DD}
**Portfolio context:** {path}
**Register:** {path}
**Critic iterations:** {N}

## Bottom line

{One paragraph. What are the top 1–3 risks to critical path, and what should leadership do about them? Be direct — this is for people who will read only this section.}

## Critical-path risks

{Bulleted — only risks with Critical or High severity AND a direct critical-path linkage. Each bullet: risk title, one-sentence description, one-sentence mitigation posture, residual risk level.}

## Secondary risks

{Medium-severity risks or risks without direct critical-path linkage. Same format, shorter.}

## Items the critic challenged but were held firm

{One line per dissent, drawn from the register's Dissents fields. Format: "R-NNN: critic argued {claim}; held because {user rationale}". This is the audit trail — not a liability, but evidence of deliberate judgment.}

## Deferred items

{Minor findings the user chose to defer, plus any Minor/Suggestion findings from the critic across all iterations. Short list, no elaboration needed.}

## Context gaps

{If the interview-stage critique flagged Critical gaps that were proceeded-past, list them here so the reader knows the limits of the assessment. If no gaps, omit this section.}
```

### Step 3: Terminal summary

Print to the terminal:

```
## Risk Assessment Complete

Portfolio context: {path}
Risk register:     {path}
Executive summary: {path}

Critic iterations: {N}
Risks identified:  {count}
Dissents logged:   {count}

Ready for review.
```

---

## Incremental Update

Entered when `/risk:assess --context <name-or-path>` points to an existing portfolio context file.

### Step 1: Load context

Read the existing portfolio context file. Display the `Last updated` date and the `Programs in scope` section to the user.

### Step 2: Ask what changed

```
This context was last updated on {date}. What's changed since then?

Paste updated documents, describe changes, or tell me what shifted — I'll update the context file and run a fresh risk analysis.
```

### Step 3: Update the context file

Merge the user's changes into the existing context file:
- Update affected fields in place
- Append a new entry to `Revision history`: `- {YYYY-MM-DD}: {one-line summary of what changed}`
- Update the `Last updated:` field

### Step 4: Re-run interview critique

Spawn `risk-critic` with `model: sonnet` in interview-sufficiency mode against the updated context file. Same flow as the fresh interview critique (max 2 loops, Critical findings go back to user, proceed if only Major/Minor).

### Step 5: Proceed to register drafting

Flow continues from **Draft Risk Register** above. A new register file is created with today's date — old registers are preserved as historical snapshots.

---

## Examples

```bash
# Fresh assessment from a timeline YAML and a strategy doc
/pm:assess phase2-timeline.yaml ~/Desktop/program-alpha-strategy.pdf

# Fresh assessment from an Excel file
/pm:assess ~/Downloads/master-timeline.xlsx

# Fresh assessment with no input files — pure interview
/pm:assess

# List existing portfolio context files
/pm:assess --list

# Re-analyze with updated context
/pm:assess --context portfolio-program-alpha-2026-04-11.md

# Re-analyze by slug (finds most recent match)
/pm:assess --context program-alpha

# Incremental update with new files
/pm:assess --context program-alpha ~/Desktop/updated-timeline.xlsx
```

---

## Notes

- The user always wins disagreements. The critic's job is to make the user think, not to overrule.
- Dissent notes are audit trail, not liability. They exist so the user can reconstruct their reasoning later.
- Progressive de-escalation is deliberate: Opus catches structural issues on the first pass; Sonnet handles residual cleanup efficiently. Interview critique is always Sonnet.
- Do not generate risks for categories that are genuinely not applicable. But do not skip categories because they are hard.
- "Monitor closely" is not a mitigation. Every mitigation must be specific enough that someone could execute it.
- The "considered and dropped" section is important. The critic will specifically check whether anything there should be promoted. Do not omit it.
