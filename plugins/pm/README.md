# pm

Program management tools for biotech PMs. Ships two skills and a browser-based timeline editor.

| Skill | What it does |
|-------|-------------|
| [`/pm:timeline`](skills/timeline/SKILL.md) | Extract, draft, and edit program timelines from prose, conversation, or structured shifts. Pairs with the [browser editor](../../tools/timeline-editor/README.md). |
| [`/pm:assess`](skills/assess/SKILL.md) | Adversarial risk assessment for drug pipeline critical path — interviews for context, drafts a structured register, then iterates with a [critic sub-agent](agents/risk-critic.md) until stress-tested. |

## Files

```
plugins/pm/
├── .claude-plugin/plugin.json
├── commands/
│   ├── timeline.md
│   └── assess.md
├── skills/
│   ├── timeline/SKILL.md
│   └── assess/SKILL.md
├── agents/
│   └── risk-critic.md
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
```
