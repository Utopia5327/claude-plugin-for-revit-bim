# claude-plugin-for-revit-bim — Claude Code Plugin for Revit & BIM

Give Claude deep knowledge of Autodesk Revit, Dynamo, BIM standards, and Autodesk Construction Cloud — so you can generate ready-to-run Dynamo Python scripts, audit models, manage parameters, detect clashes, export IFC, and automate full BIM workflows, all from plain language.

**v1.0 — built for testing and early feedback.** Issues, ideas, and feature requests are very welcome.

---

## What's inside

**25 skills** · **3 specialized agents** · **Built-in hooks**

---

## Installation

### Claude Desktop (Cowork)

Customize → Personal → **Add marketplace from GitHub** → `Utopia5327/revit-bim`

### Claude Code CLI

```
/plugin marketplace add Utopia5327/revit-bim
/plugin install revit-bim@revit-bim
```

### Local / development

```bash
claude --plugin-dir /path/to/revit-bim
```

---

## Requirements

- **Claude Code** 1.0.33+ or **Claude Desktop** with Cowork mode
- **Autodesk Revit** 2019+ (for running the generated scripts)
- **Dynamo** 2.x (IronPython 2.7) or 3.x (CPython 3) — scripts default to IronPython 2.7
- No additional dependencies — all scripts use Revit's built-in API

---

## Live Revit connection (optional)

This plugin generates scripts and provides BIM expertise out of the box. For **live two-way interaction** with a running Revit model, pair it with a Revit MCP connector:

- [**revit-mcp**](https://github.com/revit-mcp/revit-mcp) — open-source MCP server (MIT), 25+ tools for querying and modifying Revit models
- [**NonicaTab AI Connector**](https://tools.nonica.io/AIConnector) — commercial Revit add-in with 50+ MCP tools
- [**Autodesk MCP Servers**](https://www.autodesk.com/solutions/autodesk-ai/autodesk-mcp-servers) — official Autodesk MCP support

---

## Skills (25)

### Dynamo & scripting

| Skill | What it does |
|---|---|
| `generate-dynamo` | Generate complete, ready-to-run Dynamo Python scripts from plain language. The core skill. |
| `export-dynamo-file` | Generate a `.dyn` file you can open directly in Dynamo — no copy-pasting needed. |
| `revit-api-code` | Generate Revit API code in C# or Python for macros, external commands, add-ins, or pyRevit scripts. |

### Model analysis & auditing

| Skill | What it does |
|---|---|
| `audit-model` | Model health report — element counts, families, warnings, views, sheets, workset structure. |
| `query-model` | Query, filter, list, and extract data from a Revit model. |
| `analyze-rooms` | Room and space analysis — GIA/NIA by level and department, unbounded room detection, CSV export. |
| `linked-model-check` | Check status, coordinates, and health of all linked RVT and CAD files. |
| `clash-detection` | Geometric intersection check between element sets with clash reports. |

### Parameters & data

| Skill | What it does |
|---|---|
| `manage-parameters` | Read, write, and batch-update Revit element parameters. |
| `shared-parameters` | Create, bind, and manage shared parameter files and definitions. |
| `validate-parameters` | Validate required parameters are filled and within acceptable values — BEP compliance reports. |
| `batch-rename-elements` | Batch rename elements using prefixes, suffixes, numbering, or parameter-driven rules. |

### Schedules, sheets & views

| Skill | What it does |
|---|---|
| `create-schedule` | Programmatic schedule creation with configurable fields, filters, sorting, and sheet placement. |
| `manage-sheets` | Batch sheet creation, view placement, title blocks, and renumbering from Excel. |
| `create-view-filters` | Create and apply view filters with graphic overrides — color-code by parameter, discipline, etc. |

### Export & interoperability

| Skill | What it does |
|---|---|
| `export-ifc` | IFC export with configurable version (IFC 2x3 / IFC 4), property sets, and base quantities. |
| `export-to-excel` | Export element parameters, room data, schedules, or any category to CSV or Excel. |

### Model editing

| Skill | What it does |
|---|---|
| `manipulate-elements` | Move, rotate, copy, delete, mirror, and batch-edit Revit elements programmatically. |
| `workset-manager` | Audit workset assignments, find misplaced elements, and move elements between worksets. |

### Project setup

| Skill | What it does |
|---|---|
| `project-setup` | Automate project setup — worksets, levels, grids, view templates, browser organization from a BEP. |

### Autodesk Construction Cloud (ACC)

| Skill | What it does |
|---|---|
| `acc-api-setup` | Set up Python authentication and a reusable APS API client for ACC. |
| `acc-docs` | Manage ACC Docs — list folders/files, upload, download, create folder structures, transmittals. |
| `acc-coordinate` | Work with ACC Coordinate — model sets, clash testing, clash groups, coordination workflows. |
| `acc-design-collaboration` | Revit ↔ ACC Design Collaboration — cloud worksharing, design packages, shared views. |
| `acc-reports` | Export ACC data to CSV/Excel — issues, RFIs, document logs, clash summaries, activity reports. |

---

## Agents (3)

| Agent | Model | What it does |
|---|---|---|
| **BIM Agent** | Claude Opus | Expert BIM Manager and Revit specialist. Activates for Dynamo scripting, Revit API code, model auditing, clash detection, IFC export, schedule creation, parameter management, and AEC project coordination. |
| **ACC Agent** | Claude Sonnet | Autodesk Construction Cloud specialist. Handles ACC Docs, Coordinate, Design Collaboration, APS API scripting, OAuth2 authentication, and full ACC project administration. |
| **Project Setup Agent** | Claude Sonnet | Revit project configurator and BIM standards specialist. Handles new project setup, BIM Execution Plans, shared parameters, worksets, view templates, ISO 19650 compliance, and COBie data. |

---

## Hooks

Built-in safety and logging hooks:

- **PostToolUse (Write/Edit)** — logs a timestamp when files are saved
- **PostToolUse (Write)** — reminds you to test scripts on a detached copy before running on your live project
- **PreToolUse (Bash)** — logs when commands are executed

---

## How to use generated Dynamo scripts

1. Open **Dynamo** from Revit (Manage → Dynamo)
2. Create a new graph
3. Place a **Python Script** node (right-click → Search → "Python Script")
4. Double-click the node to open the editor and paste the generated code
5. Connect any required inputs (the script comments explain what each `IN[0]`, `IN[1]` expects)
6. Click **Run**

---

## Skill commands

Invoke skills explicitly:

```
/revit-bim:generate-dynamo rename all doors using level and number
/revit-bim:audit-model
/revit-bim:analyze-rooms GIA by department
/revit-bim:clash-detection structural beams vs MEP pipes
```

Or just describe what you need naturally — the BIM Agent will route your request to the right skill automatically.

---

## Contributing

Issues, pull requests, and skill suggestions welcome at:
**https://github.com/Utopia5327/revit-bim**

---

## License

MIT — see [LICENSE](LICENSE)

---

## Author

Made by [Manas Bhatia](https://github.com/Utopia5327)
For feedback: manasbhatia.design@gmail.com
