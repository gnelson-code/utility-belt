# /pm:assess

Adversarial risk assessment for drug pipeline critical path. An orchestrator interviews you for portfolio context, drafts a structured risk register, then an independent critic sub-agent stress-tests it across multiple iterations until no Critical or Major findings remain.

## How it works

```
┌─────────────────────────────────────────────────┐
│  /pm:assess  (orchestrator)                     │
│  1. Interview → portfolio context file          │
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
# Fresh assessment — starts the interview
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

**Interview before analysis.** The skill parses whatever you paste, extracts what it can, and only asks for what's missing. Critical path is mandatory; without it, the skill won't proceed. A Sonnet-level critic checks whether the context is sufficient before analysis begins.

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
