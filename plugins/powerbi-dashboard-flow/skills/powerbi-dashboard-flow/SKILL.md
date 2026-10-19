---
name: powerbi-dashboard-flow
description: Connect to a Power BI model that is open in Power BI Desktop and build a dashboard from a plain-language request. Use when the user wants to connect to Power BI, asks what is in their model, or asks for measures, a date table, or a dashboard built from a description — in English or Hebrew ("תתחבר לקובץ", "תתחבר למודל", "מה יש במודל", "תבנה לי דשבורד", "תוסיף מדדים", "תבנה טבלת תאריכים"). Covers the connect-audit-fix-build order, where each kind of change has to be written, and the save/reload rules that stop Power BI Desktop from overwriting the work.
---

# Power BI dashboard flow

Build a dashboard on a model the user has open in Power BI Desktop, working from
a plain-language brief. Follow the order below: each step depends on the one
before it, and skipping the audit is how you end up building visuals on a model
that cannot support them.

## What has to be true before you start

- Power BI Desktop is running with the project open.
- The project is saved as **`.pbip`** (Power BI Project), not `.pbix`. A `.pbix`
  is a sealed binary: the model can still be edited live, but report visuals
  cannot be authored. If the user has a `.pbix`, tell them to use
  *File > Save as > .pbip* first — that is a one-time conversion.
- The `powerbi-modeling-mcp` server is connected. If its tools are missing, the
  server failed to start; say so rather than working around it.

On the first tool call the server may require its licence agreement. Do not
accept it on the user's behalf unsolicited: show them
`https://go.microsoft.com/fwlink/?LinkId=2381247`, and call `accept_eula` only
after they have reviewed it and told you to. Acceptance is stored once per
machine, in `%LOCALAPPDATA%\Microsoft\powerbi-authoring-mcp\config.json`.

## Step 1 — Connect

Power BI Desktop opens a **different local port every launch**, so never reuse a
port from earlier in the conversation or from another session. Always discover it:

1. `connection_operations` with `operation: "ListLocalInstances"`.
2. Match the instance on `parentWindowTitle` — it is the project name. With more
   than one instance running, ask the user which file rather than guessing.
3. `connection_operations` with `operation: "Connect"` and that instance's
   `connectionString`.

Reconnecting is cheap and needed once per session. It is not an install, and
nothing about it carries over from a previous session — say so if the user
expects otherwise.

## Step 2 — Audit before you build

Never go straight from the brief to visuals. Read the model first:

- `table_operations` / `operation: "List"` — tables, column counts, storage mode
- `relationship_operations` / `operation: "List"` — the schema shape
- `measure_operations` / `operation: "List"` — what already exists
- `dax_query_operations` / `operation: "Execute"` — row counts and the date
  range, so the numbers in the dashboard can be sanity-checked

Then report what you found, and flag these four problems specifically. They are
common, they all block a usable dashboard, and the user needs to decide about
them before any visual is worth building:

| What to look for | Why it blocks a dashboard |
|---|---|
| **Zero measures** | Nothing to put in a visual; every field becomes an implicit auto-sum |
| **Hidden `LocalDateTable_*` tables** | Power BI's auto date/time, one per date column. Bloats the model and makes real time intelligence (YoY, YTD, MAT) impossible |
| **Two fact tables with identical row counts** | Usually one is a flattened copy of the other. Both joined to the same dimensions means silent double counting |
| **Several currencies in one amount column** | Summing them produces a meaningless total unless an exchange rate is applied |

Raise these as findings with a recommendation, then let the user choose. Do not
silently restructure someone's model.

## Step 3 — Foundations, then visuals

A dashboard built before the model is ready has to be rebuilt. In order:

1. **One date table**, created and marked as such via
   `table_operations` / `operation: "MarkAsDateTable"`. Then tell the user to
   turn off *Options > Current File > Time intelligence > Auto date/time* and
   that the `LocalDateTable_*` tables disappear on the next save.
   A calculated date table is created empty: `table_operations` / `"Create"`
   returns a "Refresh required" warning, and the table has no values until you
   call `RefreshWithXMLA` on it. After you then add the relationship to the fact
   table, queries still fail with *"relationship ... does not hold any data
   because it needs to be recalculated"* until you run `model_operations` /
   `RefreshWithXMLA` with `refreshType: "Calculate"`. Both refreshes are
   required, in that order — create, refresh table, relate, recalculate.
2. **Base measures** via `measure_operations` / `operation: "Create"`. Set
   `formatString` and `displayFolder` as you create them — retrofitting
   formatting across a finished dashboard is tedious. Start with the handful the
   brief actually needs (revenue, cost, margin, counts), not a speculative library.
3. **Validate** each measure with `dax_query_operations` /
   `operation: "Validate"` before relying on it, and spot-check the value with
   `Execute`. A measure that returns a plausible-looking wrong number is worse
   than one that errors.

   With several currencies in one amount column, find the rate column's
   direction before writing the measure instead of assuming it. Check the rate
   on the rows whose currency is the base one: if those rows are exactly 1, the
   column holds local units per base unit, so converting means **dividing** by
   it. Getting this backwards produces totals that look plausible and are wrong.

For the detail of model authoring — TMDL syntax, calculation groups, role-playing
dimensions — defer to the **`semantic-model-authoring`** skill rather than
improvising.

## Step 4 — Build the report

Report visuals live in PBIR JSON files on disk, under
`<project>.Report/definition/`:

```
definition/
  report.json                               theme, report-level settings
  version.json                              PBIR version
  pages/
    pages.json                              page order, active page
    <pageId>/page.json                      page size, name, filters
    <pageId>/visuals/<visualId>/visual.json one folder per visual
```

Read the canvas size from `page.json` before laying anything out — it is not
always 1280×720, and positions that overflow it are silently off-screen.

Use the **`powerbi-report-authoring`** skill for the visual JSON schema and
**`powerbi-report-design`** for layout and theming. Those carry the authoritative
schema; do not hand-write visual JSON from memory.

Translate the user's prose into a page plan first — which visuals, which fields,
what the page is answering — and show it to them before writing files. A page
plan is quick to correct; a written-out report is not.

### Hebrew and RTL dashboards

When the brief is in Hebrew, the dashboard should be too:

- Name measures and set `displayFolder` in Hebrew; they appear in the field list.
- Put the primary KPI top-**right**, not top-left, and lay the page out so it
  reads right to left. Position is the only RTL lever you have: **PBIR has no
  direction or right-to-left property**, on the page or on any visual, so do not
  go looking for one. Text direction comes from the model culture and the
  Desktop language setting, which are the user's to change, not yours. Say so
  rather than implying you set it.
- Hebrew labels need a little more width than their English equivalents — check
  the screenshot for clipping rather than assuming the default widths hold.
- Category values usually stay in whatever language the source data uses. Titles
  and measure names are yours to write in Hebrew; column values are not, so do
  not translate them silently.

## Step 5 — Making changes appear, live

Both kinds of change show up in the open Desktop window without the user
closing or reopening anything. Never tell them to reopen the project.

| Change | How it lands | What the user sees |
|---|---|---|
| Measures, date table, model metadata | Live over XMLA, through the MCP server | Appears in the field list immediately — no save, no reload |
| Report pages and visuals | PBIR files on disk, then `powerbi-desktop reload --pid <pid>` | Appears on the canvas when the reload returns |

So the loop for visuals is: write the PBIR files, run
`powerbi-report-author validate <path-to-.Report-dir>`, then
`powerbi-desktop reload --pid <pid>`. Confirm with
`powerbi-desktop screenshot <pageId> --pid <pid>` and look at the result before
claiming anything rendered.

### Write and reload in one breath — a save in between destroys the work

Between writing PBIR files and reloading, the files on disk and Desktop's
in-memory report disagree, and **Desktop's save wins**. Ctrl+S in that window
writes Desktop's older report over everything you just wrote, silently: no
error, no warning, and the validator still passes because the overwritten files
are themselves valid. The giveaway afterwards is that your edits are gone and
the `$schema` versions have changed to whatever build of Desktop did the saving.

So never pause between writing and reloading, and never ask for a save there.
Sequence every round like this:

1. `powerbi-desktop status`. If `hasUnsavedChanges` is `true`, deal with it
   **now** — ask the user to save or discard, and wait. Their unsaved UI edits
   would otherwise be lost by the reload, and their eventual save would eat
   your files.
2. Only once it is `false`: write the PBIR files.
3. `powerbi-report-author validate <path-to-.Report-dir>`.
4. `powerbi-desktop reload --pid <pid>` — immediately.
5. Screenshot and look at it.

After the reload Desktop's memory matches the files, so a later Ctrl+S is
harmless again.

This bites hardest when a model change and a report change land in the same
request — for example moving measures to a dedicated measure table, which
changes both the model *and* every `SourceRef.Entity` and `queryRef` in the
visuals that use them. Finish and save the model work first, then write the
report files and reload. Never interleave.

### What saving is actually for

Model changes made through the MCP server live in Desktop's in-memory model.
They are real and queryable at once, but they are not on disk until the user
presses Ctrl+S. If they close Desktop without saving, the measures are gone
while the PBIR files that reference them remain — and the report opens with
broken fields. So: saving is about surviving a close, not about making the
change visible. Ask for it at the end of a round of model work, and say why.

One thing that genuinely is a hazard: **never edit model TMDL files on disk
while Desktop has the project open.** Desktop holds the model in memory and
overwrites those files on its next save. Make model changes through the MCP
server instead.

When you finish, say what rendered and what is still unsaved. Do not describe a
dashboard as finished before a reload and a screenshot have confirmed it.

## Reporting back

Tell the user what is now true in their model, and be specific about what you did
not do: a measure you could not validate, a finding they have not decided on, a
visual that needs a reload before it appears. "Done" means they can see it in
Power BI Desktop, not that the files are written.
