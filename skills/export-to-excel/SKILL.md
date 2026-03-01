---
name: export-to-excel
description: >
  Export Revit model data — element parameters, room data, door/window schedules, family
  instances, or any category — to a CSV or Excel-compatible file using Dynamo Python.
  Use this skill when a user wants to get model data out of Revit into a spreadsheet for
  reporting, cost estimation, client delivery, or BIM data validation. Trigger on phrases
  like "export to Excel", "export to CSV", "get data out of Revit", "export parameters",
  "export schedule", "export room data", "extract model data", "create a data dump",
  "export family list", "export door schedule", or any request to pull Revit data into
  a tabular file format.
---

# Export Revit Model Data to Excel / CSV

Export model data to a spreadsheet for:

**"$ARGUMENTS"**

## Before writing code

Ask (one question if ambiguous):
1. **Which category?** Rooms, Doors, Windows, Walls, Families, Sheets, or a custom one?
2. **Which parameters?** List them, or default to the most useful set for that category.
3. **Output path?** Default to `C:\Exports\revit_export.csv` — user can change it.

Read-only — no model modifications. Safe to run on any model.

---

## Script: Universal parameter export to CSV

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

import csv, os, System

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ────────────────────────────────────────────────────────────
CATEGORY    = BuiltInCategory.OST_Rooms   # change to target category
OUTPUT_PATH = r"C:\Exports\revit_export.csv"

# Parameters to export — exact names, case-sensitive
# Leave empty [] to export ALL available parameters (creates wide table)
PARAMS = [
    "Name",
    "Number",
    "Level",
    "Area",
    "Department",
    "Occupancy",
    "Comments",
]
# ─────────────────────────────────────────────────────────────────────────────

def get_param_value(el, param_name):
    """Safely read any parameter as a display string."""
    param = el.LookupParameter(param_name)
    if not param:
        return ""
    storage = param.StorageType
    if storage == StorageType.String:
        return param.AsString() or ""
    elif storage == StorageType.Double:
        # Return in display units (mm² for area, mm for length)
        try:
            return str(round(param.AsDouble() * 0.0929, 3))  # ft² → m²
        except Exception:
            return str(param.AsDouble())
    elif storage == StorageType.Integer:
        return str(param.AsInteger())
    elif storage == StorageType.ElementId:
        eid = param.AsElementId()
        ref_el = doc.GetElement(eid)
        return ref_el.Name if ref_el else str(eid.IntegerValue)
    return ""

try:
    elements = list(FilteredElementCollector(doc)
                    .OfCategory(CATEGORY)
                    .WhereElementIsNotElementType()
                    .ToElements())

    if not elements:
        OUT = "No elements found in the specified category."
    else:
        # Determine headers
        if PARAMS:
            headers = ["ElementId"] + PARAMS
        else:
            # Auto-discover all parameters from first element
            headers = ["ElementId"]
            for p in elements[0].Parameters:
                if p.Definition:
                    headers.append(p.Definition.Name)

        # Ensure output directory exists
        os.makedirs(os.path.dirname(OUTPUT_PATH), exist_ok=True)

        # Write CSV
        with open(OUTPUT_PATH, 'w', newline='') as f:
            writer = csv.writer(f)
            writer.writerow(headers)

            for el in elements:
                row = [str(el.Id.IntegerValue)]
                for param_name in headers[1:]:   # skip ElementId
                    row.append(get_param_value(el, param_name))
                writer.writerow(row)

        OUT = {
            "Exported": len(elements),
            "Fields": headers,
            "File": OUTPUT_PATH
        }

except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Pre-built export recipes

### Rooms export (area in m², department, level)

```python
CATEGORY = BuiltInCategory.OST_Rooms
PARAMS   = ["Name", "Number", "Level", "Area", "Department",
            "Occupancy", "Base Finish", "Ceiling Finish", "Comments"]
```

### Door schedule export

```python
CATEGORY = BuiltInCategory.OST_Doors
PARAMS   = ["Mark", "Level", "Width", "Height", "Type Comments",
            "Frame Type", "Fire Rating", "Manufacturer", "Model"]
```

### Window schedule export

```python
CATEGORY = BuiltInCategory.OST_Windows
PARAMS   = ["Mark", "Level", "Width", "Height", "Type Comments",
            "Fire Rating", "U-Value", "SHGC"]
```

### Structural columns

```python
CATEGORY = BuiltInCategory.OST_StructuralColumns
PARAMS   = ["Mark", "Level", "Base Level", "Top Level",
            "Structural Material", "Comments"]
```

### All family instances (flat list)

```python
CATEGORY = BuiltInCategory.OST_GenericModel
PARAMS   = ["Family", "Type", "Mark", "Level", "Comments"]
```

---

## Linked models

To export data from **linked models**, add this before the collector:

```python
# Get all loaded RVT links
links = FilteredElementCollector(doc).OfClass(RevitLinkInstance).ToElements()
for link in links:
    link_doc = link.GetLinkDocument()
    if link_doc:
        elements += list(FilteredElementCollector(link_doc)
                         .OfCategory(CATEGORY)
                         .WhereElementIsNotElementType()
                         .ToElements())
```

## Unit handling

Revit stores lengths in **decimal feet** internally. Common conversions:
- Area: `ft² × 0.0929 = m²`
- Length: `ft × 304.8 = mm`

For Revit 2022+, use `UnitUtils.ConvertFromInternalUnits(value, UnitTypeId.Meters)` instead.

## Customization tips

- Change `OUTPUT_PATH` to any folder with write access (mapped drive, project server, etc.)
- For large models, add a `Level` filter: `.WhereElementIsNotElementType().ToElements()` after adding a `LevelFilter`
- Open the CSV in Excel → Data → Text to Columns if formatting looks off
