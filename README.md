# powerbi-dashboard-flow

A Claude Code plugin for building Power BI dashboards by describing them in
plain language, in English or Hebrew.

It teaches Claude the working order for a model open in Power BI Desktop:
connect, audit the model, fix the foundations, then build — plus the save and
reload rules that decide whether the work survives.

## Install

```bash
claude plugin marketplace add https://github.com/Shlomi-Unique/powerbi-course.git
```

```bash
claude plugin install powerbi-dashboard-flow@uniquetech-powerbi
```

## Requirements

- **Power BI Desktop**, with the project open while you work
- The project saved as **`.pbip`**, not `.pbix` — a `.pbix` allows model edits
  but no report authoring
- The `powerbi-authoring` plugin from
  [microsoft/skills-for-fabric](https://github.com/microsoft/skills-for-fabric),
  which provides the Power BI modeling MCP server
- Two CLIs, for validating reports and reloading them into the running Desktop:

```bash
npm install -g @microsoft/powerbi-report-authoring-cli @microsoft/powerbi-desktop-bridge-cli
```

## Use

With the project open in Power BI Desktop, ask in your own words:

> connect to the open Power BI model and show me what's in it

> build a date table and mark it

> add base measures — revenue, cost, margin

> build a dashboard page with revenue by month, by category and by store

Model changes appear in the field list immediately. Report changes land when
Claude reloads the report into Desktop — you never close and reopen the file.

## What the skill covers

- Finding the Desktop instance, whose local port changes every launch
- Auditing a model for the four problems that block a usable dashboard: no
  measures, auto date/time tables, duplicated fact tables, and mixed currencies
- The two refreshes a calculated date table needs before it holds data
- Which changes go live over XMLA and which go to PBIR files on disk
- Why a save in the wrong moment silently overwrites report files
- Laying out a Hebrew dashboard, and what RTL in PBIR can and cannot do

## Licence

MIT — see [LICENSE](LICENSE).
