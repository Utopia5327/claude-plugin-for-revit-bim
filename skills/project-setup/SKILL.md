---
name: project-setup
description: >
  Automate Revit project setup — create worksets, levels, grids, view templates, browser
  organization schemes, and standard views from a project brief or BIM Execution Plan.
  Use this skill at project start when a user needs to set up a new Revit model from
  scratch or bring an existing model up to BIM standards. Trigger on phrases like
  "project setup", "set up the model", "create worksets", "add levels", "set up grids",
  "create view templates", "BEP setup", "model template", "create standard views",
  "set up browser organisation", "new project setup", "initialise the model", or any
  request to configure a Revit model for a new project. Also offer this skill when a
  user complains their model doesn't have proper worksets or is missing standard views.
---

# Revit Project Setup Automation

Set up the Revit model for:

**"$ARGUMENTS"**

## Before writing code

Ask (one question if ambiguous):
1. **What project type?** Residential, commercial, mixed-use, infrastructure?
2. **Which components?** Worksets, levels, grids, view templates, standard views — or all?
3. **Any specific names/numbers?** (e.g. level heights, grid spacing, workset names from BEP)

Each script below is standalone — run only what's needed.

---

## Script 1: Create worksets from a list

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ─────────────────────────────────────────────────────────
WORKSET_NAMES = [
    "Architecture",
    "Structure",
    "MEP",
    "Site",
    "Interiors",
    "Shared Levels and Grids",
    "Linked Models",
    "CAD Links",
]
# ─────────────────────────────────────────────────────────────────────────

try:
    if not doc.IsWorkshared:
        OUT = "ERROR: This model is not workshared. Enable worksharing first (Collaborate > Worksets)."
    else:
        existing = [ws.Name for ws in
                    FilteredWorksetCollector(doc).OfKind(WorksetKind.UserWorkset).ToWorksets()]
        created = []
        skipped = []

        TransactionManager.Instance.EnsureInTransaction(doc)

        for name in WORKSET_NAMES:
            if name not in existing:
                Workset.Create(doc, name)
                created.append(name)
            else:
                skipped.append(name + " (already exists)")

        TransactionManager.Instance.TransactionTaskDone()
        OUT = {"Created": created, "Skipped": skipped}

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Script 2: Create levels at specified heights

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ─────────────────────────────────────────────────────────
# (name, height in mm above datum)
LEVELS = [
    ("00 Ground Floor",   0),
    ("01 First Floor",    3600),
    ("02 Second Floor",   7200),
    ("03 Third Floor",    10800),
    ("04 Fourth Floor",   14400),
    ("Roof Level",        18000),
]

MM_TO_FT = 1.0 / 304.8   # Revit stores lengths in decimal feet
# ─────────────────────────────────────────────────────────────────────────

try:
    existing_names = [l.Name for l in
                      FilteredElementCollector(doc).OfClass(Level).ToElements()]
    created = []
    skipped = []

    TransactionManager.Instance.EnsureInTransaction(doc)

    for level_name, height_mm in LEVELS:
        if level_name in existing_names:
            skipped.append(level_name + " (already exists)")
            continue

        height_ft = height_mm * MM_TO_FT
        new_level = Level.Create(doc, height_ft)
        new_level.Name = level_name
        created.append(level_name + " @ " + str(height_mm) + " mm")

    TransactionManager.Instance.TransactionTaskDone()
    OUT = {"Created levels": created, "Skipped": skipped}

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Script 3: Create a grid layout (orthogonal)

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ─────────────────────────────────────────────────────────
# Vertical grids (along X axis) — positions in mm from origin
VERTICAL_GRID_POSITIONS = [0, 6000, 12000, 18000, 24000]
VERTICAL_GRID_LABELS    = ["A", "B", "C", "D", "E"]

# Horizontal grids (along Y axis) — positions in mm from origin
HORIZONTAL_GRID_POSITIONS = [0, 7500, 15000, 22500]
HORIZONTAL_GRID_LABELS    = ["1", "2", "3", "4"]

GRID_LENGTH_MM = 30000   # total length of grid lines
MM_TO_FT = 1.0 / 304.8
# ─────────────────────────────────────────────────────────────────────────

def ft(mm):
    return mm * MM_TO_FT

try:
    created_grids = []
    TransactionManager.Instance.EnsureInTransaction(doc)

    # Vertical grids (run top to bottom, i.e. in Y direction)
    for i, x_mm in enumerate(VERTICAL_GRID_POSITIONS):
        x = ft(x_mm)
        start = XYZ(x, ft(-2000), 0)
        end   = XYZ(x, ft(GRID_LENGTH_MM), 0)
        line  = Line.CreateBound(start, end)
        grid  = Grid.Create(doc, line)
        label = VERTICAL_GRID_LABELS[i] if i < len(VERTICAL_GRID_LABELS) else str(i+1)
        grid.Name = label
        created_grids.append("Vertical " + label + " @ X=" + str(x_mm) + "mm")

    # Horizontal grids (run left to right, i.e. in X direction)
    for j, y_mm in enumerate(HORIZONTAL_GRID_POSITIONS):
        y = ft(y_mm)
        start = XYZ(ft(-2000), y, 0)
        end   = XYZ(ft(GRID_LENGTH_MM), y, 0)
        line  = Line.CreateBound(start, end)
        grid  = Grid.Create(doc, line)
        label = HORIZONTAL_GRID_LABELS[j] if j < len(HORIZONTAL_GRID_LABELS) else str(j+1)
        grid.Name = label
        created_grids.append("Horizontal " + label + " @ Y=" + str(y_mm) + "mm")

    TransactionManager.Instance.TransactionTaskDone()
    OUT = {"Grids created": created_grids, "Total": len(created_grids)}

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Script 4: Create standard floor plan views for all levels

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ─────────────────────────────────────────────────────────
# View types to create per level
VIEW_SUFFIXES = ["_ARCH", "_STRUCT", "_MEP"]
VIEW_FAMILY   = ViewFamily.FloorPlan    # or ViewFamily.CeilingPlan
# ─────────────────────────────────────────────────────────────────────────

try:
    levels     = list(FilteredElementCollector(doc).OfClass(Level).ToElements())
    view_types = list(FilteredElementCollector(doc).OfClass(ViewFamilyType).ToElements())
    fp_type    = next((vt for vt in view_types if vt.ViewFamily == VIEW_FAMILY), None)

    if not fp_type:
        OUT = "ERROR: No Floor Plan view family type found in document."
    else:
        existing_view_names = {v.Name for v in
                               FilteredElementCollector(doc).OfClass(View).ToElements()}
        created = []

        TransactionManager.Instance.EnsureInTransaction(doc)

        for level in levels:
            for suffix in VIEW_SUFFIXES:
                view_name = level.Name + suffix
                if view_name not in existing_view_names:
                    new_view = ViewPlan.Create(doc, fp_type.Id, level.Id)
                    new_view.Name = view_name
                    created.append(view_name)

        TransactionManager.Instance.TransactionTaskDone()
        OUT = {"Views created": created, "Total": len(created)}

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Recommended setup sequence

Tell the user to run scripts in this order:
1. **Script 1** — Create worksets (model must already be workshared)
2. **Script 2** — Create levels
3. **Script 3** — Create grids
4. **Script 4** — Create standard views per level
5. Then: use `manage-sheets` skill to create the drawing register

## Safety notes

- Workset creation requires a workshared model. If not workshared, instruct the user to go to Collaborate → Worksets first.
- Level and grid creation modifies the model — recommend running on a fresh model or a detached copy.
- View names must be unique — the scripts check for duplicates before creating.
