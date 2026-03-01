---
name: linked-model-check
description: >
  Check the status, coordinates, and health of all RVT and CAD linked files in a Revit model.
  Generates a coordination report showing which links are loaded, unloaded, missing, or have
  coordinate mismatches. Use this skill before coordination meetings, model submissions, or
  federated model reviews. Trigger on phrases like "linked model", "check links", "RVT links",
  "CAD links", "link status", "coordinate check", "shared coordinates", "model federation",
  "are all links loaded", "missing links", "linked file health", "unloaded links", "link report",
  or any request to inspect the status of files linked into a Revit model. Also offer this
  proactively before IFC export or Navisworks federation.
---

# Linked Model Health Check

Audit linked files and coordination status for:

**"$ARGUMENTS"**

## Approach

Default to a comprehensive read-only report covering:
- All RVT links (loaded, unloaded, missing, not found)
- All CAD imports/links (DWG, DXF, DGN)
- Coordinate system status (shared coordinates, survey point)
- Linked model bounding boxes (detect obvious coordinate outliers)

---

## Script: Comprehensive linked file report (read-only)

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

import csv, os

doc  = DocumentManager.Instance.CurrentDBDocument
MM_TO_FT = 1.0 / 304.8

def ft_to_mm(ft_val):
    return round(ft_val * 304.8, 1)

try:
    report = {}

    # ── 1. RVT Links ──────────────────────────────────────────────────────────
    rvt_link_types = list(FilteredElementCollector(doc)
                          .OfClass(RevitLinkType).ToElements())
    rvt_link_instances = list(FilteredElementCollector(doc)
                              .OfClass(RevitLinkInstance).ToElements())

    rvt_report = []
    for lt in rvt_link_types:
        load_state = lt.GetLinkedFileStatus()
        path_info  = lt.GetExternalFileReference()
        path_str   = ""
        try:
            path_str = ModelPathUtils.ConvertModelPathToUserVisiblePath(
                path_info.GetAbsolutePath())
        except Exception:
            path_str = "Path unavailable"

        # Find matching instances
        instances = [li for li in rvt_link_instances
                     if li.GetTypeId() == lt.Id]

        for inst in instances:
            xform = inst.GetTransform()
            origin = xform.Origin
            link_doc = inst.GetLinkDocument()

            entry = {
                "Name":           lt.Name,
                "Status":         str(load_state),
                "Path":           path_str,
                "Instance Count": len(instances),
                "Origin X (mm)":  ft_to_mm(origin.X),
                "Origin Y (mm)":  ft_to_mm(origin.Y),
                "Origin Z (mm)":  ft_to_mm(origin.Z),
                "Is Identity":    xform.IsIdentity,
                "Linked Doc":     link_doc.Title if link_doc else "NOT LOADED",
            }

            # Coordinate warning
            if abs(origin.X) > 328 or abs(origin.Y) > 328:  # > 100m offset
                entry["WARNING"] = "Link origin > 100m from project origin — check shared coordinates"
            else:
                entry["WARNING"] = ""

            rvt_report.append(entry)

    # If no instances (type exists but not placed)
    for lt in rvt_link_types:
        instances = [li for li in rvt_link_instances if li.GetTypeId() == lt.Id]
        if not instances:
            rvt_report.append({
                "Name":     lt.Name,
                "Status":   str(lt.GetLinkedFileStatus()),
                "Path":     "Type exists but not placed as instance",
                "WARNING":  "No instance placed in model"
            })

    report["RVT Links"] = rvt_report

    # ── 2. CAD Links & Imports ────────────────────────────────────────────────
    cad_items = list(FilteredElementCollector(doc)
                     .OfClass(ImportInstance).ToElements())
    cad_report = []
    for ci in cad_items:
        try:
            cat_name = ci.Category.Name if ci.Category else "Unknown"
            is_linked = ci.IsLinked
            xform     = ci.GetTransform()
            origin    = xform.Origin

            entry = {
                "Category":      cat_name,
                "Is Linked":     is_linked,
                "Origin X (mm)": ft_to_mm(origin.X),
                "Origin Y (mm)": ft_to_mm(origin.Y),
                "Origin Z (mm)": ft_to_mm(origin.Z),
            }
            try:
                ext_ref = ci.GetExternalFileReference()
                entry["Path"] = ModelPathUtils.ConvertModelPathToUserVisiblePath(
                    ext_ref.GetAbsolutePath())
            except Exception:
                entry["Path"] = "Embedded / no path"

            cad_report.append(entry)
        except Exception:
            pass

    report["CAD Links/Imports"] = cad_report

    # ── 3. Coordinate summary ─────────────────────────────────────────────────
    # Project Base Point and Survey Point
    base_pts   = list(FilteredElementCollector(doc)
                      .OfClass(BasePoint)
                      .ToElements())
    coord_info = []
    for bp in base_pts:
        try:
            is_survey = bp.IsShared
            pos       = bp.Position
            coord_info.append({
                "Type":       "Survey Point" if is_survey else "Project Base Point",
                "X (mm)":     ft_to_mm(pos.X),
                "Y (mm)":     ft_to_mm(pos.Y),
                "Z (mm)":     ft_to_mm(pos.Z),
            })
        except Exception:
            pass

    report["Coordinate Points"] = coord_info

    # ── 4. Summary ───────────────────────────────────────────────────────────
    loaded   = sum(1 for r in rvt_report
                   if "LinkedFileStatus_Loaded" in str(r.get("Status", "")))
    unloaded = sum(1 for r in rvt_report
                   if "LinkedFileStatus_Loaded" not in str(r.get("Status", ""))
                   and r.get("Status"))
    warnings = sum(1 for r in rvt_report if r.get("WARNING"))

    report["Summary"] = {
        "Total RVT Links":   len(rvt_link_types),
        "Instances placed":  len(rvt_link_instances),
        "Loaded":            loaded,
        "Unloaded/Missing":  unloaded,
        "Coordinate Warnings": warnings,
        "CAD Items":         len(cad_items),
    }

    OUT = report

except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Script: Reload all unloaded RVT links

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

try:
    link_types = list(FilteredElementCollector(doc)
                      .OfClass(RevitLinkType).ToElements())
    reloaded = []
    failed   = []

    TransactionManager.Instance.EnsureInTransaction(doc)

    for lt in link_types:
        status = lt.GetLinkedFileStatus()
        if str(status) != "LinkedFileStatus_Loaded":
            try:
                result = lt.Reload()
                if str(result) == "LinkLoadResultType_Success":
                    reloaded.append(lt.Name)
                else:
                    failed.append(lt.Name + " (" + str(result) + ")")
            except Exception as ex:
                failed.append(lt.Name + " (Exception: " + str(ex) + ")")

    TransactionManager.Instance.TransactionTaskDone()
    OUT = {"Reloaded": reloaded, "Failed": failed}

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Interpreting the report

Guide the user on key red flags:

- **Status ≠ Loaded**: find the file on the server and use Manage Links → Reload From to fix path
- **Origin > 100m from project origin**: shared coordinates are likely wrong — check Survey Point vs Project Base Point
- **Is Identity = False**: the link has been moved from its reference point — may cause coordination issues
- **CAD import (not linked)**: imported DWGs bloat the model — recommend converting to linked CAD
- **No instance placed**: the link type exists in the file but wasn't placed — it's still taking up file size

## What to tell the user

Always summarise:
1. How many links are loaded vs unloaded
2. Any coordinate warnings
3. Whether any CAD files are imported (not linked) — these should be converted to links
4. Next steps: who owns each link and needs to fix their model
