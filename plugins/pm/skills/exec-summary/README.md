# /pm:exec-summary

Concision editor for executive communications. Takes a draft document, surfaces analyst questions, produces a structured exec-ready rewrite, then runs adversarial critique from the reader's perspective.

## How it works

```
┌─────────────────────────────────────────────────┐
│  /pm:exec-summary  (orchestrator)               │
│  1. Ingest draft → identify audience            │
│  2. Analyst questions (confusion = signal)       │
│  3. Structured rewrite                          │
│  4. Critic loop (Opus → Sonnet, 2 rounds)       │
│  5. Finalize → notes/exec/                      │
└──────────────────┬──────────────────────────────┘
                   │ spawns each round
                   ▼
┌─────────────────────────────────────────────────┐
│  concision-critic  (sub-agent)                  │
│  • Recommendation clarity test                  │
│  • Standalone test (can I decide without the    │
│    original?)                                   │
│  • Jargon detection for target audience         │
│  • Impact specificity — qualitative → quantify  │
│  • Length / density check (~400 words target)    │
│  • Hedging removal                              │
│  • Missing alternatives                         │
└─────────────────────────────────────────────────┘
```

## Usage

```bash
# Rewrite a draft document
/pm:exec-summary ~/Desktop/migration-proposal.md

# Specify the audience
/pm:exec-summary proposal.pdf --audience "C-suite, non-technical"

# Multiple source files
/pm:exec-summary proposal.md cost-analysis.xlsx

# No files — paste content directly
/pm:exec-summary
```

## Key design decisions

**Your confusion is signal.** Before rewriting, the orchestrator reads the draft as someone with roughly exec-level context. Places where it's confused map to places the exec will be confused. Things that feel unnecessary are probably cuttable. Both signals are surfaced as questions before the rewrite begins.

**Two modes.** The skill detects whether the draft is asking for a decision or informing for awareness, and picks the right structure. Decision mode: Problem → Why it matters → Recommendation → Key context → What we considered. Awareness mode: Bottom line → What happened → What this means → What's next. Sections can be dropped if empty or added if the content demands it. Target length: under 2 minutes of reading time (~400 words).

**Progressive de-escalation.** Opus runs the first critic pass (catches structural issues — unclear recommendations, missing context, jargon). Sonnet handles round 2 (residual cleanup). Default is 2 rounds.

**Escalation on request.** If the user asks for more rounds after the default 2, the loop resets to Opus and de-escalates again. This repeats as many times as the user needs. The logic: if the default rounds weren't enough, the remaining issues are hard enough to warrant Opus again.

**No audit trail.** Unlike `/pm:assess`, dismissed critic findings are not logged. Exec summaries don't need a dissent record — if you disagree with the critic, you just move on.

## Artifacts

Outputs go to `notes/exec/`:

| File | Purpose |
|------|---------|
| `{slug}-{YYYY-MM-DD}.md` | Final exec summary with YAML frontmatter (source, audience, date, rounds) |

## Rewrite principles

- Lead with the ask — the reader knows what you want by the end of section 2
- No jargon without inline translation
- Quantify impact — "$200K/quarter" not "significant savings"
- Cut hedging — unless the uncertainty is genuine and load-bearing
- Preserve nuance that matters — "works if X" is different from "works"
- Every bullet in Key Context must pass: "Would removing this change the reader's decision?"
