# revit-bim — Claude Code Plugin

Connect Claude to Autodesk Revit and the BIM workflow. Generate Dynamo Python scripts, query model data, detect clashes, audit model health, manage parameters, create schedules, export IFC, and automate full BIM workflows — all from plain English.

---

## What it does

Once installed, Claude becomes a BIM-aware assistant that can:

| Skill | What you can ask |
|---|---|
| `generate-dynamo` | *"Write a Dynamo script that renames all rooms using Level-Number format"* |
| `audit-model` | *"Audit my model — how many warnings, families, and views does it have?"* |
| `query-model` | *"List all walls with a Fire Rating of 2HR on Level 3"* |
| `analyze-rooms` | *"Give me a GIA breakdown by department with a list of any unbounded rooms"* |
| `manage-parameters` | *"Batch-set the 'Specification' parameter on all doors from this Excel list"* |
| `clash-detection` | *"Check structural columns against MEP ducts for geometric clashes"* |
| `create-schedule` | *"Create a door schedule with Mark, Type, Width, Height, and From/To Room fields"* |
| `export-ifc` | *"Export the model to IFC 2x3 with base quantities to C:\Projects\IFC"* |
| `revit-api-code` | *"Write a pyRevit script that isolates rooms by department in the active view"* |
| `manage-sheets` | *"Create sheets A-001 through A-020 using my title block, from this Excel list"* |
| `manipulate-elements` | *"Move all furniture on Level 2 up by 150mm to account for a raised floor"* |

All scripts produced are ready to paste into a Dynamo Python Script node and run.

---

## Installation

### From the official marketplace (recommended)

In Claude Code:
```
/plugin install revit-bim
```

### From GitHub

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

- **Claude Code** 1.0.33 or later
- **Autodesk Revit** 2019 or later (for running the scripts this plugin generates)
- **Dynamo** 2.x (IronPython 2.7) or Dynamo 3.x (CPython) — scripts default to IronPython 2.7, which is compatible with most firms
- No additional dependencies — all scripts use Revit's built-in API

---

## How to use the generated Dynamo scripts

1. Open **Dynamo** from Revit (Manage → Dynamo)
2. Create a new graph
3. Place a **Python Script** node (right-click → Search → "Python Script")
4. Double-click the node to open the editor and paste the generated code
5. Connect any required inputs (the script comments explain what each `IN[0]`, `IN[1]` expects)
6. Click **Run**

---

## Skills overview

### `generate-dynamo`
The core skill. Generates complete, runnable Dynamo Python scripts for any Revit automation task. Defaults to IronPython 2.7 for maximum compatibility. Every script includes:
- Complete imports and boilerplate
- Transaction handling (where the model is modified)
- try/except error handling
- `OUT = result` at the end
- Generous inline comments

### `audit-model`
Read-only model health report covering element counts by category, total families, in-place families, total views, sheets, model warnings, and workset structure.

### `query-model`
Filtered element collection and data extraction. Produces read-only scripts that list, count, and extract model data — safe to run on any project.

### `analyze-rooms`
Room and space analysis: GIA/NIA by level and department, space program comparison, unbounded/unplaced room detection, and CSV export.

### `manage-parameters`
Read and write Revit element parameters in bulk. Handles type detection (String/Double/Integer), read-only checks, and batch updates from Excel data.

### `clash-detection`
Geometric intersection check between two sets of elements using Boolean operations. Produces clash reports with element IDs, types, and clash volumes.

### `create-schedule`
Programmatic schedule creation with configurable fields, filters, sorting, and sheet placement.

### `export-ifc`
IFC export with configurable version (IFC 2x3 / IFC 4), space boundaries, base quantities, and linked file options.

### `revit-api-code`
C# external commands, Revit macros, .addin manifests, and pyRevit button scripts. For when you need persistent tools beyond one-off Dynamo graphs.

### `manage-sheets`
Batch sheet creation, view placement, view template application, and sheet renumbering from Excel.

### `manipulate-elements`
Move, rotate, mirror, copy, and change the type of Revit elements programmatically. Always warns to test on a backup first.

---

## Skill commands

Once installed, you can invoke skills explicitly:

```
/revit-bim:generate-dynamo rename all doors using level and number
/revit-bim:audit-model
/revit-bim:analyze-rooms GIA by department
/revit-bim:clash-detection structural beams vs MEP pipes
```

Or just describe what you need naturally — the `bim-agent` that activates with this plugin will route your request to the right skill automatically.

---

## Contributing

Issues, pull requests, and skill suggestions welcome at:
**https://github.com/Utopia5327/revit-bim**

Skill ideas we'd love to add in future releases:
- Workset management and element assignment
- Revision cloud tracking
- Family parameter batch editor
- Linked model coordination reporting
- COBie data export

---

## License

MIT — see [LICENSE](LICENSE)

---

## Author

Made by [Manas Bhatia](https://github.com/Utopia5327)
For feedback: manasbhatia.design@gmail.com
