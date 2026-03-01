---
name: workset-manager
description: >
  Audit workset assignments across a Revit model, identify elements on the wrong workset,
  and move elements between worksets using Dynamo Python. Use this skill when checking
  model workset structure before submission, finding walls on the MEP workset, moving
  elements to the correct workset based on category rules, generating a workset
  assignment report, or cleaning up workset mistakes before issuing. Trigger on phrases
  like "workset", "wrong workset", "move to workset", "workset audit", "workset report",
  "workset assignment", "elements on wrong workset", "workset cleanup", "check worksets",
  "workset structure", or any request involving workset management and auditing.
---

# Workset Manager — Audit & Fix

Audit and manage worksets for:

**"$ARGUMENTS"**

## Before writing code

Ask (one question if ambiguous):
1. **Audit or fix?** Just report which elements are on wrong worksets, or also move them?
2. **What are the workset rules?** (e.g. Walls/Floors/Roofs → Architecture, Ducts/Pipes → MEP)
3. **Scope?** Entire model, or one level/phase?

Always run the audit script first, then the move script if needed.

---

## Script 1: Workset audit report (read-only)

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

import csv, os

doc = DocumentManager.Instance.CurrentDBDocument

try:
    if not doc.IsWorkshared:
        OUT = "This model is not workshared — worksets are not applicable."
    else:
        # Get all user worksets
        worksets = {ws.Id: ws.Name for ws in
                    FilteredWorksetCollector(doc)
                    .OfKind(WorksetKind.UserWorkset)
                    .ToWorksets()}

        # Categories to audit and their expected workset (partial match OK)
        EXPECTED_WORKSETS = {
            "Walls":                  "Architecture",
            "Floors":                 "Architecture",
            "Roofs":                  "Architecture",
            "Ceilings":               "Architecture",
            "Stairs":                 "Architecture",
            "Railings":               "Architecture",
            "Curtain Panels":         "Architecture",
            "Structural Columns":     "Structure",
            "Structural Framing":     "Structure",
            "Structural Foundations": "Structure",
            "Duct Curves":            "MEP",
            "Pipe Curves":            "MEP",
            "Conduits":               "MEP",
            "Cable Trays":            "MEP",
            "Mechanical Equipment":   "MEP",
            "Plumbing Fixtures":      "MEP",
            "Lighting Fixtures":      "MEP",
        }

        BIC_MAP = {
            "Walls":                  BuiltInCategory.OST_Walls,
            "Floors":                 BuiltInCategory.OST_Floors,
            "Roofs":                  BuiltInCategory.OST_Roofs,
            "Ceilings":               BuiltInCategory.OST_Ceilings,
            "Stairs":                 BuiltInCategory.OST_Stairs,
            "Railings":               BuiltInCategory.OST_StairsRailing,
            "Curtain Panels":         BuiltInCategory.OST_CurtainWallPanels,
            "Structural Columns":     BuiltInCategory.OST_StructuralColumns,
            "Structural Framing":     BuiltInCategory.OST_StructuralFraming,
            "Structural Foundations": BuiltInCategory.OST_StructuralFoundation,
            "Duct Curves":            BuiltInCategory.OST_DuctCurves,
            "Pipe Curves":            BuiltInCategory.OST_PipeCurves,
            "Conduits":               BuiltInCategory.OST_Conduit,
            "Cable Trays":            BuiltInCategory.OST_CableTray,
            "Mechanical Equipment":   BuiltInCategory.OST_MechanicalEquipment,
            "Plumbing Fixtures":      BuiltInCategory.OST_PlumbingFixtures,
            "Lighting Fixtures":      BuiltInCategory.OST_LightingFixtures,
        }

        misplaced = []
        summary   = {}

        for cat_name, bic in BIC_MAP.items():
            expected_partial = EXPECTED_WORKSETS.get(cat_name, "")
            elements = list(FilteredElementCollector(doc)
                            .OfCategory(bic)
                            .WhereElementIsNotElementType()
                            .ToElements())
            wrong = 0
            for el in elements:
                ws_param = el.get_Parameter(BuiltInParameter.ELEM_PARTITION_PARAM)
                if not ws_param:
                    continue
                ws_id   = WorksetId(ws_param.AsInteger())
                ws_name = worksets.get(ws_id, "Unknown")
                if expected_partial and expected_partial.lower() not in ws_name.lower():
                    misplaced.append({
                        "Category":        cat_name,
                        "ElementId":       str(el.Id.IntegerValue),
                        "Current Workset": ws_name,
                        "Expected":        expected_partial,
                    })
                    wrong += 1

            summary[cat_name] = {
                "Total":     len(elements),
                "Misplaced": wrong,
                "Expected":  expected_partial,
            }

        OUT = {
            "Summary":          summary,
            "Misplaced count":  len(misplaced),
            "Misplaced (first 50)": misplaced[:50],
            "Available worksets": list(worksets.values()),
        }

except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Script 2: Move elements to correct worksets by category rule

**⚠ This modifies the model. Test on a detached copy first.**

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ────────────────────────────────────────────────────────────
# Maps BuiltInCategory → target workset name (partial match)
REASSIGNMENT_RULES = [
    (BuiltInCategory.OST_Walls,              "Architecture"),
    (BuiltInCategory.OST_Floors,             "Architecture"),
    (BuiltInCategory.OST_Roofs,              "Architecture"),
    (BuiltInCategory.OST_Ceilings,           "Architecture"),
    (BuiltInCategory.OST_StructuralColumns,  "Structure"),
    (BuiltInCategory.OST_StructuralFraming,  "Structure"),
    (BuiltInCategory.OST_DuctCurves,         "MEP"),
    (BuiltInCategory.OST_PipeCurves,         "MEP"),
    (BuiltInCategory.OST_MechanicalEquipment,"MEP"),
    (BuiltInCategory.OST_PlumbingFixtures,   "MEP"),
    (BuiltInCategory.OST_LightingFixtures,   "MEP"),
]
# ─────────────────────────────────────────────────────────────────────────────

try:
    if not doc.IsWorkshared:
        OUT = "This model is not workshared."
    else:
        # Build workset lookup: name fragment → WorksetId
        all_worksets = {ws.Name: ws.Id for ws in
                        FilteredWorksetCollector(doc)
                        .OfKind(WorksetKind.UserWorkset)
                        .ToWorksets()}

        def find_workset_id(partial_name):
            for ws_name, ws_id in all_worksets.items():
                if partial_name.lower() in ws_name.lower():
                    return ws_id
            return None

        moved   = []
        skipped = []

        TransactionManager.Instance.EnsureInTransaction(doc)

        for bic, target_name in REASSIGNMENT_RULES:
            target_id = find_workset_id(target_name)
            if not target_id:
                skipped.append("Workset matching '" + target_name + "' not found")
                continue

            elements = list(FilteredElementCollector(doc)
                            .OfCategory(bic)
                            .WhereElementIsNotElementType()
                            .ToElements())

            for el in elements:
                ws_param = el.get_Parameter(BuiltInParameter.ELEM_PARTITION_PARAM)
                if ws_param and not ws_param.IsReadOnly:
                    current_id = WorksetId(ws_param.AsInteger())
                    if current_id != target_id:
                        ws_param.Set(target_id.IntegerValue)
                        moved.append(str(el.Id.IntegerValue) + " → " + target_name)

        TransactionManager.Instance.TransactionTaskDone()
        OUT = {
            "Moved":   len(moved),
            "Skipped": skipped,
            "Details (first 50)": moved[:50]
        }

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Script 3: List all worksets and element counts per workset (read-only)

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

try:
    if not doc.IsWorkshared:
        OUT = "Not a workshared model."
    else:
        worksets = list(FilteredWorksetCollector(doc)
                        .OfKind(WorksetKind.UserWorkset)
                        .ToWorksets())

        # Count elements per workset
        ws_counts = {ws.Id: {"Name": ws.Name, "Count": 0, "Owner": ws.Owner or "Not checked out"}
                     for ws in worksets}

        all_elements = list(FilteredElementCollector(doc)
                            .WhereElementIsNotElementType()
                            .ToElements())

        for el in all_elements:
            ws_param = el.get_Parameter(BuiltInParameter.ELEM_PARTITION_PARAM)
            if ws_param:
                ws_id = WorksetId(ws_param.AsInteger())
                if ws_id in ws_counts:
                    ws_counts[ws_id]["Count"] += 1

        result = sorted(ws_counts.values(), key=lambda x: -x["Count"])
        OUT = result

except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Workset best practices to share with users

- **Shared Levels and Grids**: always keep levels and grids on a dedicated workset — never on Architecture or Structure
- **Linked Models**: all `RevitLinkInstance` objects should go on a "Linked Models" workset
- **Ownership**: remind users to relinquish all worksets after a session (Collaborate → Relinquish All)
- **Naming**: use consistent names across projects — "Architecture" not "Arch", "ARCH", or "01-Architecture"
