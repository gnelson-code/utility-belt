# /pm:assess

Adversarial risk assessment for drug pipeline critical path. An orchestrator interviews you for portfolio context, drafts a structured risk register, then an independent critic sub-agent stress-tests it across multiple iterations until no Critical or Major findings remain.

## How it works

```
┌─────────────────────────────────────────────────┐
│  /pm:assess  (orchestrator)                     │
│  1. Analyze input files → interview gaps →      │
│     portfolio context file                      │
│  2. Draft risk register                         │
│  3. Critic loop (Opus → Sonnet, max 4 rounds)  │
│  4. Executive summary                           │
└──────────────────┬──────────────────────────────┘
                   │ spawns each iteration
                   ▼
┌─────────────────────────────────────────────────┐
│  risk-critic  (sub-agent)                       │
│  • Attacks severity scores and rationale        │
│  • Challenges critical-path linkages            │
│  • Checks mitigations actually mitigate         │
│  • Surfaces missed risk categories              │
│  • Reviews "considered and dropped" for errors  │
└─────────────────────────────────────────────────┘
```

## Usage

```bash
# Seed from a timeline YAML and a strategy doc
/pm:assess phase2-timeline.yaml ~/Desktop/program-alpha-strategy.pdf

# Seed from an Excel file
/pm:assess ~/Downloads/master-timeline.xlsx

# No input files — pure interview
/pm:assess

# List existing portfolio context files
/pm:assess --list

# Re-analyze with updated context (incremental — asks "what changed?")
/pm:assess --context portfolio-program-alpha-2026-04-11.md

# Match by slug (finds most recent)
/pm:assess --context program-alpha
```

## Key design decisions

**User always wins.** When the critic challenges a risk and you disagree, you win. The critic's objection is logged as a dissent on that risk entry — an audit trail of what was challenged and why you held firm.

**Files first, interview second.** Hand it timelines (Excel, YAML, JSON), strategy docs, or PDFs — the skill analyzes them and extracts programs, stages, timelines, dependencies, and critical-path signals. The interview only asks for what the files left open. If you already have a `/pm:timeline` YAML, most of the interview is pre-filled. Critical path is mandatory; without it, the skill won't proceed. A Sonnet-level critic checks whether the context is sufficient before analysis begins.

**Progressive de-escalation.** Opus runs the first critic pass on the register (catches structural gaps, missed categories, bad severity scoring). Sonnet handles iterations 2–4 (residual cleanup). Hard cap at 4 iterations.

**Persistence.** Portfolio context files are saved to `notes/risk/` and reusable. Running `--context` against an existing file asks what changed since last time rather than re-interviewing from scratch.

## Artifacts

All outputs go to `notes/risk/`:

| File | Purpose |
|------|---------|
| `portfolio-{slug}-{date}.md` | Structured portfolio context — programs, critical path, dependencies |
| `register-{slug}-{date}.md` | Risk register — the source of truth, with dissent log |
| `exec-summary-{slug}-{date}.md` | Executive summary — produced only after all iterations complete |

## Deferred

- Rotating multi-persona critic (regulatory / clinical / CMC / commercial)
- Quantitative risk scoring (probabilistic, Monte Carlo)
- Integration with external trackers (Smartsheet, JIRA)
- Longitudinal tracking across assessments
- Export to PDF / PowerPoint / Word
