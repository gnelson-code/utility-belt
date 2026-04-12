# pm

Program management tools for biotech PMs. Ships three skills and a browser-based timeline editor.

| Skill | What it does |
|-------|-------------|
| [`/pm:timeline`](skills/timeline/SKILL.md) | Extract, draft, and edit program timelines from prose, conversation, or structured shifts. Pairs with the [browser editor](../../tools/timeline-editor/README.md). |
| [`/pm:assess`](skills/assess/SKILL.md) | Adversarial risk assessment for drug pipeline critical path — interviews for context, drafts a structured register, then iterates with a [critic sub-agent](agents/risk-critic.md) until stress-tested. |
| [`/pm:exec-summary`](skills/exec-summary/SKILL.md) | Concision editor for executive communications — takes a draft, surfaces analyst questions, produces a structured exec-ready rewrite, then runs adversarial critique via [concision-critic](agents/concision-critic.md). |

## Supported inputs

All three skills accept files as positional arguments. The file format table below applies to all of them.

### File formats

| Extension | How it's read |
|-----------|--------------|
| `.xlsx`, `.xls` | Parsed as spreadsheet — columns matched by synonym (Task/Activity, Start Date/Begin, Status/State, etc.) |
| `.yaml`, `.yml` | Parsed as structured data — if it matches the `/pm:timeline` canonical schema (`workstreams` + `tasks` keys), fields are extracted directly |
| `.json` | Same as YAML — parsed as structured data |
| `.md`, `.txt` | Read as unstructured prose — programs, dates, milestones, dependencies, and risk language extracted by analysis |
| `.pdf` | Read as unstructured prose (extractable text only) |

### Content types

The file format tells the skill *how* to parse; the content tells it *what* to extract. Common content types:

| Content | What the skills extract from it |
|---------|-------------------------------|
| **Program timeline** (Excel or YAML from `/pm:timeline`) | Programs, workstreams, tasks, dates, statuses, milestone structure. `/pm:assess` also infers stage and dependencies from task sequencing. |
| **Strategy / war doc** | Program names, approval pathways, regulatory authority, competitive context, strategic rationale. Rich source of critical-path "why." |
| **Steering committee deck / meeting notes** | Status updates, recently surfaced risks, decision log, action items. Often contains implicit critical-path signals ("we need X before Y"). |
| **Regulatory submission plan** | Approval pathway, submission timeline, agency interactions, required studies. Directly maps to regulatory risk category. |
| **Prior risk register / assessment** | Previously identified risks, severity scores, mitigations, critical-path descriptions. Baseline for incremental updates. |
| **Protocol or study report** | Endpoints, enrollment targets, comparator design, safety monitoring. Maps to clinical data risk category. |

Multiple files can be provided at once — the skills cross-reference them (e.g., a timeline gives dates while a strategy doc gives rationale).

## Files

```
plugins/pm/
├── .claude-plugin/plugin.json
├── commands/
│   ├── timeline.md
│   ├── assess.md
│   └── exec-summary.md
├── skills/
│   ├── timeline/SKILL.md
│   ├── assess/SKILL.md
│   └── exec-summary/SKILL.md
├── agents/
│   ├── risk-critic.md
│   └── concision-critic.md
└── README.md

tools/timeline-editor/          # paired browser tool for /pm:timeline
├── index.html
├── sample-timeline.yaml
├── examples/
│   └── sample-program.xlsx
└── README.md

notes/risk/                     # output directory for /pm:assess artifacts
├── portfolio-*.md              # portfolio context files
├── register-*.md               # risk registers
└── exec-summary-*.md           # executive summaries

notes/exec/                     # output directory for /pm:exec-summary artifacts
└── {slug}-{date}.md            # exec summaries
```
