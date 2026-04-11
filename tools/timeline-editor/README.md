# Timeline Editor

A zero-setup visual editor for program timelines. No install, no terminal, no signup.

## Open It

Double-click `index.html`. It opens in your default browser. That's it.

If double-clicking opens it in a text editor instead, right-click → **Open With** → Google Chrome (or Safari, Firefox, or Edge).

## Load a Timeline

You have three ways to load data:

1. **Drag and drop an Excel file** onto the page.
2. **Drag and drop a YAML file** onto the page (the format this tool uses internally).
3. Click the **Open File** button and pick one.

If you don't load anything, the page starts with a sample timeline so you can see how it works.

### Excel file requirements

Your Excel file should have these columns (column names are flexible — see below):

| What we need | Examples of column names we accept |
|---|---|
| Task name | Task, Task Name, Name, Activity, Project, Project Name |
| Workstream / group | Workstream, Group, Category, Swimlane, Team, Function |
| Start date | Start, Start Date, Begin |
| End date (or duration) | End, End Date, Finish, Duration, Days |
| Status | Status, State |

**Milestones:** any row with a duration of 0 (or the same start and end date) becomes a diamond milestone.

**Status values we recognize:** Not Started, In Progress, Complete, At Risk, Blocked, Delayed, Estimate (variations in capitalization and punctuation are fine).

## Edit

- **Change any value** — click in the table on the left. Dates use a date picker, status and workstream are dropdowns.
- **Add a task** — click the **+ Task** button.
- **Delete a task** — click the × on the task row.
- **Add a workstream** — click **+ Workstream**.
- **Collapse a workstream** — click its header in the timeline on the right.

Everything updates live. There is no save button and no undo — if you want to keep your work, export it.

## Export

Three export buttons at the top:

- **Download PNG** — a high-resolution image. Drop this straight into a PowerPoint slide.
- **Download SVG** — a vector version that stays sharp at any size. Also works in PowerPoint.
- **Download YAML** — the data in the format this tool uses internally. Save this if you want to come back and edit later, or if you want Claude to apply bulk changes via `/pm:timeline`.

## Coming back to edit later

1. Open the timeline editor the same way as before.
2. Drop your saved YAML file onto the page.
3. Your timeline appears exactly as you left it.

## If something goes wrong

- **The page is blank.** Make sure you opened `index.html` in a browser, not a text editor.
- **My Excel file won't load.** The error banner at the top tells you what's missing. The most common issue is a missing or misnamed column — check that you have Task, Workstream, Start, and End (or Duration).
- **I closed the tab by accident and lost my work.** There is no autosave. Next time, click **Download YAML** before closing.
- **The colors look wrong.** Each status has a fixed color. Blocked is pink. If something looks wrong, check that the status values in your Excel file match the list above.
