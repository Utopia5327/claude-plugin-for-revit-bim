---
name: clash-detection
description: >
  Detect and report geometric clashes and interferences between Revit model elements using
  Dynamo Python and the Revit API's Boolean intersection tools. Use this skill when checking
  for MEP vs Structure clashes, discipline coordination, overlapping element checks, or any
  geometric interference between two sets of elements. Trigger on phrases like "clash detection",
  "interference check", "find clashes", "MEP coordination", "structural clash", "duct vs beam",
  "pipe vs column", "coordination report", or any request to find elements that overlap or
  intersect in 3D space. Also offer this skill when users mention coordination meetings or
  model submission reviews.
---

# Clash Detection & Coordination

Perform clash detection for:

**"$ARGUMENTS"**

## Before scripting

Clarify:
1. **Which two disciplines** are you checking? (e.g. structural columns vs MEP ducts)
2. **How should clashes be reported?** (list of IDs, CSV export, isolate in view?)
3. **Minimum clash volume?** Default threshold is very small (1e-6 ft³) to catch all contacts.

Note: for large models, Boolean intersection on every element pair is slow. Recommend running on a specific level or workset, or using Navisworks for production-scale clash detection.

## Clash detection script

Read-only — no model modifications.

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Define the two element sets to check ────────────────────────────────────
# Modify these categories for your discipline combination:
set_a = list(FilteredElementCollector(doc)
             .OfCategory(BuiltInCategory.OST_StructuralColumns)
             .WhereElementIsNotElementType()
             .ToElements())

set_b = list(FilteredElementCollector(doc)
             .OfCategory(BuiltInCategory.OST_DuctCurves)
             .WhereElementIsNotElementType()
             .ToElements())

# ── Helper: extract solid geometry from an element ──────────────────────────
def get_solid(element):
    opts = Options()
    opts.ComputeReferences = False
    opts.DetailLevel = ViewDetailLevel.Fine
    geo = element.get_Geometry(opts)
    if geo is None:
        return None
    for obj in geo:
        if isinstance(obj, Solid) and obj.Volume > 1e-9:
            return obj
        if isinstance(obj, GeometryInstance):
            for sub in obj.GetInstanceGeometry():
                if isinstance(sub, Solid) and sub.Volume > 1e-9:
                    return sub
    return None

# ── Run clash detection ──────────────────────────────────────────────────────
clashes = []

try:
    for el_a in set_a:
        solid_a = get_solid(el_a)
        if solid_a is None:
            continue

        # Get type name once per el_a (minor optimisation)
        type_id_a = el_a.GetTypeId()
        type_name_a = (doc.GetElement(type_id_a).Name
                       if type_id_a != ElementId.InvalidElementId else 'N/A')

        for el_b in set_b:
            solid_b = get_solid(el_b)
            if solid_b is None:
                continue
            try:
                intersection = BooleanOperationsUtils.ExecuteBooleanOperation(
                    solid_a, solid_b, BooleanOperationsType.Intersect)

                if intersection and intersection.Volume > 1e-6:
                    type_id_b = el_b.GetTypeId()
                    type_name_b = (doc.GetElement(type_id_b).Name
                                   if type_id_b != ElementId.InvalidElementId else 'N/A')
                    clashes.append({
                        'Element A ID':   el_a.Id.IntegerValue,
                        'Element A Type': type_name_a,
                        'Element B ID':   el_b.Id.IntegerValue,
                        'Element B Type': type_name_b,
                        'Clash Volume (ft3)': round(intersection.Volume, 6),
                        'Clash Volume (cm3)': round(intersection.Volume * 28316.8, 2),
                    })
            except:
                # Boolean operation failed — elements may share a face (contact, not clash)
                pass

    OUT = [len(clashes), clashes]

except Exception as e:
    import traceback
    OUT = 'ERROR: ' + str(e) + '\n' + traceback.format_exc()
```

## Isolating clashing elements in a view

Once you have the clash IDs, show the user how to isolate them:

```python
# IN[0] = list of element IDs from the clash report (integers)
# This script isolates clashing elements in the active view
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc   = DocumentManager.Instance.CurrentDBDocument
uidoc = DocumentManager.Instance.CurrentUIApplication.ActiveUIDocument
view  = uidoc.ActiveView

clash_ids = [ElementId(int(i)) for i in IN[0]]

TransactionManager.Instance.EnsureInTransaction(doc)
view.IsolateElementsTemporary(clash_ids)
TransactionManager.Instance.TransactionTaskDone()
OUT = 'Isolated ' + str(len(clash_ids)) + ' elements in active view'
```

## Exporting clash report to CSV

```python
import csv
output_path = r'C:\Reports\clash_report.csv'
if clashes:
    with open(output_path, 'w') as f:
        writer = csv.DictWriter(f, fieldnames=clashes[0].keys())
        writer.writeheader()
        writer.writerows(clashes)
    OUT = 'Clash report saved: ' + output_path + ' (' + str(len(clashes)) + ' clashes)'
```

## Performance note

For models with thousands of elements, the pairwise Boolean check can be slow. Recommend:
- Running on a single level at a time (use `ElementLevelFilter`)
- Using Autodesk Navisworks for production-scale coordination
- This script is best for targeted checks (e.g. new structural additions vs existing MEP)
