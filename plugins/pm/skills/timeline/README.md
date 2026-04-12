# /pm:timeline

Extract, draft, and edit program timelines. Pairs with the [browser editor](../../../../tools/timeline-editor/README.md) — Claude handles reasoning, the editor handles visual layout and export.

## How the two surfaces work together

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

| If you want to… | Use |
|---|---|
| Turn a protocol PDF or meeting notes into a timeline | `/pm:timeline` |
| Build a new timeline from a conversation | `/pm:timeline` |
| Shift a whole phase of the program | `/pm:timeline --shift-from` |
| Bulk-mark tasks as blocked / at-risk / delayed | `/pm:timeline --edit` |
| Import an Excel file | the editor (drag-drop) |
| Tweak a task's name, dates, or status visually | the editor |
| Export a PNG for a slide | the editor |

## Usage

```
/pm:timeline [input-path-or-yaml] [--output <yaml>] [--edit "<instruction>"] [--shift-from <task> --by <duration>]
```

### Three modes

**1. Prose extraction.** Point it at a document and Claude reads it for tasks, dates, owners, and statuses. Anything ambiguous gets surfaced as a numbered question list before the YAML is written — the skill won't silently guess at dates.

```
/pm:timeline ~/Desktop/phase2-protocol.pdf
/pm:timeline meeting-notes.md --output program.yaml
```

**2. Conversational draft.** No file, no flags. Claude runs a short interview (program name, timeframe, workstreams, major milestones) and produces a starter YAML.

```
/pm:timeline
```

**3. Edit.** Point it at an existing YAML and either describe the change in English or use the structured shift form.

```
/pm:timeline program.yaml --edit "mark IND writing as blocked and add a legal review milestone on Sept 1"
/pm:timeline program.yaml --shift-from "IND submission" --by 6w
```

`--shift-from` matches by task ID, exact name, or substring. `--by` takes a signed duration (`14d`, `6w`, `-3m`). Tasks starting on or after the anchor shift; earlier tasks are locked. Durations are preserved exactly.

### What it does NOT do

- Parse Excel — drop `.xlsx` files onto the editor instead.
- Invent tasks — if your source doesn't mention it, Claude won't add it.
- Renumber task IDs on edits — IDs are stable across round-trips.
- Render timelines in the terminal — it prints a `file://` URL for the editor.

## Canonical schema

Both surfaces read and write this YAML shape. Status enum is fixed at seven values.

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
    start: 2026-01-15
    end: 2026-04-30
    status: in_progress   # not_started | in_progress | complete | at_risk | blocked | delayed | estimate
    milestone: false
```

## Typical workflow

```
# Extract a timeline from kickoff notes
/pm:timeline ~/Notes/phase2-kickoff.md --output phase2.yaml

# CMC slips by 6 weeks
/pm:timeline phase2.yaml --shift-from "Process validation" --by 6w

# New status — IND is blocked
/pm:timeline phase2.yaml --edit "mark IND writing blocked, add a legal review milestone on 2026-09-01"
```

## Deferred

- Dependency arrows between tasks
- Round-trip back to Excel
- Multi-sheet workbook import (first sheet only)
- `localStorage` autosave in the editor
- Critical-path computation
- Resource loading / utilization views
