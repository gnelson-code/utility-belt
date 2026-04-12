# pm

Program management tools for biotech PMs. Currently ships one skill (`/pm:timeline`) and one paired browser tool (`tools/timeline-editor/`).

## What it's for

Biotech program managers spend hours on two painful tasks:

1. **Building and revising program timelines** that have to end up in PowerPoint.
2. **Sifting through prose** — protocols, meeting notes, status emails, slide decks — to extract what's actually happening and when.

This plugin attacks both. The **editor** is a single HTML file for visual timeline work; the **skill** is Claude doing the reasoning work that clicking around in a chart can't.

## The two surfaces

```
┌────────────────────────────────────────────┐
│  /pm:timeline  (Claude skill)              │
│  • Extract timelines from prose documents  │
│  • Draft from a conversation               │
│  • Bulk edits in natural language          │
│  • Structured shifts: "shift X by 6 weeks" │
└──────────────────┬─────────────────────────┘
                   │ writes / reads
                   ▼
           ┌────────────────┐
           │ timeline.yaml  │  ← canonical source of truth
           └────────┬───────┘
                    │ loads / exports
                    ▼
┌────────────────────────────────────────────┐
│  tools/timeline-editor/index.html          │
│  • Drag-drop Excel / YAML / JSON           │
│  • Visual editing (tables + Gantt)         │
│  • Export PNG / SVG / YAML                 │
└────────────────────────────────────────────┘
```

**Rule of thumb:**

| If you want to… | Use |
|---|---|
| Turn a protocol PDF or meeting notes into a timeline | `/pm:timeline` |
| Build a new timeline from a conversation | `/pm:timeline` |
| Shift a whole phase of the program | `/pm:timeline --shift-from` |
| Bulk-mark tasks as blocked / at-risk / delayed | `/pm:timeline --edit` |
| Import an Excel file | the editor (drag-drop) |
| Tweak a task's name, dates, or status visually | the editor |
| Export a PNG for a slide | the editor |

The two surfaces round-trip through `timeline.yaml`, so you can move between them freely.

## The skill: `/pm:timeline`

```
/pm:timeline [input-path-or-yaml] [--output <yaml>] [--edit "<instruction>"] [--shift-from <task> --by <duration>]
```

### Three modes

**1. Prose extraction.** Point it at a document and Claude reads it for tasks, dates, owners, and statuses. Works on markdown, plain text, and PDFs that have extractable text. Anything ambiguous gets surfaced back as a numbered question list before the YAML is written — the skill won't silently guess at dates.

```
/pm:timeline ~/Desktop/phase2-protocol.pdf
/pm:timeline meeting-notes.md --output program.yaml
```

**2. Conversational draft.** No file, no flags. Claude runs a short interview (program name, timeframe, workstreams, major milestones, one question per workstream) and produces a usable starter YAML. Aim for 4–6 questions, not 20.

```
/pm:timeline
```

**3. Edit.** Point it at an existing YAML and either describe the change in English or use the structured shift form.

```
# Free-form
/pm:timeline program.yaml --edit "mark IND writing as blocked and add a legal review milestone on Sept 1"

# Structured shift — most common bulk edit
/pm:timeline program.yaml --shift-from "IND submission" --by 6w
```

`--shift-from` takes an anchor (matched by task ID, exact name, or substring), `--by` takes a signed duration (`14d`, `6w`, `-3m`). Tasks that start on or after the anchor shift by the duration; tasks that started earlier are locked in place. Durations are preserved exactly — nothing gets squeezed or stretched.

### What it does NOT do

- **It does not parse Excel.** If you hand it an `.xlsx` file it will tell you to drop the file onto the editor instead. One importer, not two.
- **It does not invent tasks.** If your source document doesn't mention something, Claude won't add it to "round out" a workstream.
- **It does not renumber task IDs on edits.** Task IDs are stable identity across round-trips with the editor.
- **It does not render timelines in the terminal.** After writing the YAML it prints a clickable `file://` URL for the editor and stops.

## The editor: `tools/timeline-editor/index.html`

Single self-contained HTML file. Libraries (React, js-yaml, SheetJS, Babel) load from CDN on first open. No `npm install`, no build step, no terminal. Double-click the file — it opens in your default browser. See [`tools/timeline-editor/README.md`](../../tools/timeline-editor/README.md) for the non-engineer walkthrough.

**What it handles:**

- Drag-drop import: `.xlsx`, `.xls`, `.yaml`, `.yml`, `.json`
- Excel column synonyms (Task / Activity / Project Name, Workstream / Group / Swimlane, Start Date / Begin, End Date / Finish / Duration, Status / State)
- Milestone inference: rows with `duration == 0` become diamonds
- Status normalization: `"In Progress"` / `"in-progress"` / `"InProgress"` all map to the canonical `in_progress`
- Collapsible workstream groups
- In-place editing (dates, status, milestone flag, task name, add/delete)
- Export PNG (2× DPR), SVG (vector), and YAML

## Canonical schema

Both surfaces read and write this YAML shape. Status enum is fixed at seven values — no free-form status strings allowed.

```yaml
title: "Phase 2 Clinical Program"
generated: 2026-04-11
workstreams:
  - id: cmc
    name: CMC
    collapsed: false
tasks:
  - id: t-001
    name: "Process validation"
    workstream: cmc
    start: 2026-01-15         # ISO YYYY-MM-DD
    end: 2026-04-30
    status: in_progress       # not_started | in_progress | complete | at_risk | blocked | delayed | estimate
    milestone: false
```

## Typical end-to-end workflow

```
# Monday: extract a timeline from the kickoff meeting notes
/pm:timeline ~/Notes/phase2-kickoff.md --output phase2.yaml

# → Claude asks about 3 ambiguous dates, you reply "ok"
# → phase2.yaml written
# → cmd+click the file:// URL to open the editor
# → drop phase2.yaml, review visually, make a few tweaks
# → Download PNG, drop into slide for tomorrow's SteerCo

# Wednesday: CMC tells you they're slipping by 6 weeks
/pm:timeline phase2.yaml --shift-from "Process validation" --by 6w

# → 9 CMC tasks shifted, everything earlier untouched
# → reopen the editor, export a fresh PNG

# Friday: new status from Regulatory — IND is now blocked
/pm:timeline phase2.yaml --edit "mark IND writing blocked, add a legal review milestone on 2026-09-01"
```

## Files

```
plugins/pm/
├── .claude-plugin/plugin.json        # plugin manifest
├── commands/timeline.md              # /pm:timeline dispatcher
├── skills/timeline/SKILL.md          # full skill body
└── README.md                         # this file

tools/timeline-editor/
├── index.html                        # the editor (single file, self-contained)
├── sample-timeline.yaml              # 12-task example, all 7 statuses
├── examples/
│   └── sample-program.xlsx           # fake biotech program for testing
└── README.md                         # non-engineer walkthrough
```

## Deferred

- Dependency arrows between tasks
- Round-trip back to Excel
- Multi-sheet workbook import (first sheet only)
- `localStorage` autosave in the editor — you must click Download YAML to persist edits
- Critical-path computation
- Resource loading / utilization views
