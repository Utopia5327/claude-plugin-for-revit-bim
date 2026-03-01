---
name: validate-parameters
description: >
  Validate that required Revit element parameters are filled in, non-empty, and within
  acceptable values — then generate a compliance report. Use this skill for BEP (BIM
  Execution Plan) data validation, pre-submission QA checks, COBie data readiness,
  checking that all rooms have departments assigned, all doors have fire ratings, or
  any "are all my parameters filled in?" question. Trigger on phrases like "validate
  parameters", "check parameters", "missing parameters", "BEP compliance", "data
  completeness", "parameter QA", "are all fields filled", "COBie validation", "pre-
  submission check", "parameter audit", "empty parameters", or any request to verify
  that element data is complete and correct.
---

# Validate Revit Parameters

Run a BIM data completeness check for:

**"$ARGUMENTS"**

## Before writing code

Ask (one question if ambiguous):
1. **Which category and which parameters** must be filled? (e.g. all Rooms must have Name, Number, Department, Occupancy)
2. **What counts as valid?** Non-empty? A value from a specific list? A number above zero?
3. **Output format?** Dynamo OUT list, or also export to CSV?

Read-only — no model modifications.

---

## Script: Required parameter completeness check

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

import csv, os

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ────────────────────────────────────────────────────────────
# Define which categories and which parameters are REQUIRED
VALIDATION_RULES = {
    BuiltInCategory.OST_Rooms: [
        "Name", "Number", "Department", "Occupancy"
    ],
    BuiltInCategory.OST_Doors: [
        "Mark", "Fire Rating", "Width", "Height"
    ],
    BuiltInCategory.OST_Windows: [
        "Mark", "Width", "Height"
    ],
}

# Optional: allowed value lists (leave empty dict {} to skip)
ALLOWED_VALUES = {
    # "Fire Rating": ["30", "60", "90", "120", "FD30", "FD60", "FD90"],
    # "Occupancy": ["Office", "Meeting", "Circulation", "WC", "Stair"],
}

# Export report to CSV? Set to None to skip file export.
OUTPUT_CSV = r"C:\Exports\parameter_validation_report.csv"
# ─────────────────────────────────────────────────────────────────────────────

def get_str_value(el, param_name):
    """Return parameter value as string, or None if missing/unset."""
    param = el.LookupParameter(param_name)
    if not param:
        return None  # Parameter doesn't exist on this element
    st = param.StorageType
    if st == StorageType.String:
        v = param.AsString()
        return v if v else ""
    elif st == StorageType.Double:
        v = param.AsDouble()
        return str(round(v, 4)) if v is not None else ""
    elif st == StorageType.Integer:
        return str(param.AsInteger())
    elif st == StorageType.ElementId:
        eid = param.AsElementId()
        ref = doc.GetElement(eid)
        return ref.Name if ref else ""
    return ""

try:
    all_issues = []
    summary = {}

    for category, required_params in VALIDATION_RULES.items():
        elements = list(FilteredElementCollector(doc)
                        .OfCategory(category)
                        .WhereElementIsNotElementType()
                        .ToElements())

        cat_name = doc.Settings.Categories.get_Item(category).Name
        issues = []
        passed = 0

        for el in elements:
            el_issues = []
            el_id = str(el.Id.IntegerValue)

            # Try to get a display name for the element
            name_param = el.LookupParameter("Name") or el.LookupParameter("Mark")
            el_name = (name_param.AsString() if name_param else None) or el_id

            for param_name in required_params:
                value = get_str_value(el, param_name)

                if value is None:
                    el_issues.append(param_name + ": PARAMETER NOT FOUND")
                elif value.strip() == "":
                    el_issues.append(param_name + ": EMPTY")
                elif param_name in ALLOWED_VALUES:
                    allowed = ALLOWED_VALUES[param_name]
                    if value not in allowed:
                        el_issues.append(param_name + ": '" + value +
                                         "' not in allowed list " + str(allowed))

            if el_issues:
                issues.append({
                    "Category": cat_name,
                    "ElementId": el_id,
                    "Element": el_name,
                    "Issues": " | ".join(el_issues)
                })
            else:
                passed += 1

        summary[cat_name] = {
            "Total": len(elements),
            "Passed": passed,
            "Failed": len(issues),
            "Pass Rate": (str(round(100 * passed / len(elements))) + "%") if elements else "N/A"
        }
        all_issues.extend(issues)

    # Write CSV report
    if OUTPUT_CSV and all_issues:
        os.makedirs(os.path.dirname(OUTPUT_CSV), exist_ok=True)
        with open(OUTPUT_CSV, 'w', newline='') as f:
            writer = csv.DictWriter(f, fieldnames=["Category", "ElementId", "Element", "Issues"])
            writer.writeheader()
            writer.writerows(all_issues)

    OUT = {
        "Summary": summary,
        "Total Issues": len(all_issues),
        "Issues": all_issues[:50],   # show first 50 in Dynamo
        "Report": OUTPUT_CSV if OUTPUT_CSV else "No file export configured"
    }

except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Pre-built validation profiles

### BEP-standard room data

```python
VALIDATION_RULES = {
    BuiltInCategory.OST_Rooms: [
        "Name", "Number", "Department", "Occupancy",
        "Base Finish", "Ceiling Finish", "Wall Finish", "Comments"
    ]
}
```

### COBie-ready asset data (doors/equipment)

```python
VALIDATION_RULES = {
    BuiltInCategory.OST_Doors: [
        "Mark", "Type Comments", "Fire Rating", "Manufacturer", "Model",
        "Warranty Duration Parts", "Warranty Duration Labor"
    ],
    BuiltInCategory.OST_MechanicalEquipment: [
        "Mark", "Manufacturer", "Model", "Serial Number", "Warranty Duration Parts"
    ]
}
```

### Structural submission check

```python
VALIDATION_RULES = {
    BuiltInCategory.OST_StructuralColumns: ["Mark", "Structural Material", "Comments"],
    BuiltInCategory.OST_StructuralFraming:  ["Mark", "Structural Material", "Cut Length"],
    BuiltInCategory.OST_StructuralFoundation: ["Mark", "Structural Material"],
}
```

---

## Interpreting results

Guide the user on what to do with failures:

- **PARAMETER NOT FOUND**: the parameter may not be bound to this category — check Shared Parameters
- **EMPTY**: element exists but data hasn't been filled in — filter by ElementId in Revit to locate them
- **Not in allowed list**: data is filled but doesn't conform to the project standard — may need a lookup table

## Output

Always tell the user:
- Pass rate per category (e.g. "Rooms: 48/52 passed — 92%")
- The top 3 most common failing parameters
- Location of the CSV report if generated
