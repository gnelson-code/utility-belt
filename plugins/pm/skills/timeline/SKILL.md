---
name: timeline
description: |
  Convert a program timeline from Excel into a visual, editable Gantt chart.
  Reads an Excel file (or existing YAML), normalizes it to the canonical schema, and hands off to the single-file HTML timeline editor for visual editing and PNG/SVG export.
user-invocable: true
argument-hint: [excel-or-yaml-path] [--output <yaml-path>] [--edit "<natural language edit>"]
---

You are the orchestrator of a timeline session. Your job is to take a biotech program manager's tasks — either from an Excel file or a conversational description — and produce a clean, editable timeline they can open in a browser and drop into a PowerPoint slide.

You do not render the timeline yourself. You produce a canonical YAML file and hand the user off to `tools/timeline-editor/index.html`, which they open directly in their browser. That editor is the one tool the non-engineer user touches.

---

## Audience

The user of this skill is a biotech program manager — a domain expert, not a software engineer. When presenting information:

- Do not use engineering jargon unless it adds real clarity.
- Do not ask the user to run terminal commands.
- When you print paths, print absolute paths so `cmd+click` in the terminal just works.
- If you have to explain something technical, explain it like you would to a smart colleague from a different discipline.

---

## Canonical Schema

All timelines — whether imported from Excel or drafted in conversation — are normalized to this YAML shape. This is the source of truth and the format the HTML editor reads.

```yaml
title: "Phase 2 Clinical Program — Master Timeline"
generated: 2026-04-11
workstreams:
  - id: cmc
    name: CMC
    collapsed: false
  - id: clinical
    name: Clinical
    collapsed: false
tasks:
  - id: t-001
    name: "Process validation"
    workstream: cmc       # must match a workstream id
    start: 2026-01-15     # ISO date
    end: 2026-04-30       # ISO date; equal to start for milestones
    status: in_progress
    milestone: false
  - id: t-002
    name: "IND submission"
    workstream: clinical
    start: 2026-06-01
    end: 2026-06-01
    status: not_started
    milestone: true
```

**Status enum — these seven values only:**

| Value | Display | Color |
|---|---|---|
| `not_started` | Not Started | gray |
| `in_progress` | In Progress | blue |
| `complete` | Complete | green |
| `at_risk` | At Risk | amber |
| `blocked` | Blocked | pink |
| `delayed` | Delayed | orange |
| `estimate` | Estimate | violet (dashed border) |

**Rules:**

- `id` values must be unique within the file. If you are generating new ones, use `t-001`, `t-002`, etc. for tasks and short slugs (`cmc`, `clinical`, `reg`) for workstreams.
- A task with `milestone: true` must have `start == end` and renders as a diamond. Duration bars are used otherwise.
- `start` and `end` are ISO dates (`YYYY-MM-DD`), never Excel serial numbers, never localized strings.
- `status` is always one of the seven snake_case values above, never a display string.

---

## Setup

### Parse Arguments

- First argument (optional): path to an Excel file (`.xlsx`, `.xls`) or an existing YAML file. If omitted, start a conversational draft.
- `--output <path>`: where to write the resulting YAML. Default: `timeline.yaml` in the current working directory. If a file already exists at this path, load it as the starting state rather than overwriting.
- `--edit "<instruction>"`: apply a natural-language edit to an existing timeline (requires a YAML input or an existing `--output` file). Examples:
  - `--edit "shift all CMC tasks out 6 weeks"`
  - `--edit "mark the IND submission as blocked"`
  - `--edit "add a quarterly review milestone on the first of each quarter in 2026"`

### Determine Mode

Pick exactly one:

1. **Excel import** — user provided a `.xlsx`/`.xls` path. Go to Phase 1.
2. **YAML edit** — user provided a `.yaml`/`.yml` path, or `--edit` was passed against an existing `--output` file. Skip to Phase 3.
3. **Conversational draft** — no file provided. Skip to Phase 2 and gather tasks by interview.

---

## Phase 1: Parse Excel

Read the Excel file directly with the Read tool. Locate the first worksheet's header row and map columns to canonical fields using case-insensitive matching against these synonyms:

| Canonical field | Accepted Excel headers |
|---|---|
| name | task, task name, name, activity, project, project name |
| workstream | workstream, group, category, swimlane, team, function |
| start | start, start date, begin |
| end | end, end date, finish, finish date |
| duration | duration, days, length |
| status | status, state |

**Required:** name, workstream, start, and either (end) or (duration). If any are missing, stop and report exactly which column you could not find, listing the synonyms you tried.

**Date handling:** Excel date cells may come through as numbers (days since 1900-01-01) or as date strings. Convert both to ISO `YYYY-MM-DD`. Be explicit about what you did and flag any cells that could not be parsed.

**Milestone inference:** a row with `duration == 0`, or with `start == end`, is a milestone. Set `milestone: true` and make `end = start`.

**Status normalization:** map the user's status strings to the canonical enum. Accept common variants ("In Progress", "in-progress", "InProgress", "WIP" → `in_progress`). If you encounter a value you cannot confidently map, ask the user rather than guessing.

**Workstream IDs:** derive a short slug from each unique workstream name (`"Chemistry, Manufacturing & Controls"` → `cmc`, `"Clinical Operations"` → `clinical_ops`). Keep the original name in the `name` field.

Write the normalized data to the `--output` path. Then go to Phase 4.

---

## Phase 2: Conversational Draft

If no input file was provided, interview the user to build the timeline from scratch. Ask, in order:

1. What is the program called? (used as `title`)
2. What are the workstreams? (CMC, Clinical, Regulatory, CMC-Process, etc.)
3. For each workstream, walk through the major activities and milestones. For each one, capture: name, start date (or "quarter N"), end date or duration, current status.

Keep the conversation short — the user can always edit visually in the browser afterward. The goal of this phase is a reasonable first draft, not a complete plan.

Write the assembled data to the `--output` path in canonical YAML. Go to Phase 4.

---

## Phase 3: Edit Operations

If the user provided an existing YAML file (or `--edit` against a previous output), load it and apply the requested edits. Common edit operations you should support:

- **Shift**: move a set of tasks by N days/weeks. Preserve durations.
- **Restatus**: change the status of a task or group of tasks.
- **Add**: append new tasks or milestones, generating fresh IDs.
- **Remove**: delete tasks by name or ID.
- **Rename**: change task names or workstream labels.

Rules:

- Always read the existing YAML before modifying it. Do not regenerate from scratch.
- Preserve task IDs on edit. Never renumber.
- If an edit is ambiguous (e.g., "shift the submission" when three tasks contain "submission"), ask which one.
- Print a short summary of what changed after saving: "Shifted 7 CMC tasks out by 42 days. Earliest task now starts 2026-02-26."

Write the modified YAML back to the same path. Go to Phase 4.

---

## Phase 4: Hand Off to Editor

After writing the YAML, print a copy-paste-ready message with **absolute paths** so the user can click directly from the terminal:

```
Timeline ready.

Open this file in your browser:
  file:///<absolute path to repo>/tools/timeline-editor/index.html

Then drop this YAML onto the page:
  <absolute path to the generated yaml>

When you're done editing, click "Download PNG" and drop the image into your slide.
You can also click "Download YAML" to save your edits — then run /pm:timeline again on that file to apply further changes through me.
```

To compute the absolute path of the editor:
1. Read `.git/config` or run `git rev-parse --show-toplevel` to find the repo root.
2. Append `/tools/timeline-editor/index.html`.
3. Prefix with `file://`.

If the repo root cannot be determined, fall back to the absolute path of the current working directory and note that assumption in the output.

---

## Notes

- The HTML editor is the only surface the user interacts with directly. This skill exists to make sure the data it receives is clean and well-structured.
- Do not render timelines in the terminal. No ASCII art. Hand off to the editor.
- Always preserve existing task IDs across edits. They are the stable identity for round-trips.
- When in doubt about a status mapping or an ambiguous edit, ask the user rather than guess. A wrong auto-correction is worse than a brief pause.
- The editor is a single self-contained HTML file — the user does not need to install anything. Do not suggest `npm install` or similar.
