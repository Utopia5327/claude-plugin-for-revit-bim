---
name: query-model
description: >
  Query, filter, list, and extract data from a Revit model database using Dynamo Python.
  Use this skill when a user wants to list elements, count elements by category, filter by
  parameter value, find elements meeting a condition, extract data for a report, or
  answer "how many X are in my model" type questions. Trigger on phrases like "list all",
  "filter elements", "find all walls", "get all rooms", "extract data", "how many",
  "show me all elements where", or any request to inspect or report on model content
  without modifying it. This skill produces read-only query scripts — safe on any model.
---

# Query Revit Model

Write a Dynamo Python script to query the Revit model for:

**"$ARGUMENTS"**

## Read-only query script template

No transaction needed — this script only reads data, it doesn't modify the model.

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

try:
    # ── Build a filtered element collector ──────────────────────────────────
    # Modify the category or class filter to match what you need:
    collector = (FilteredElementCollector(doc)
                 .OfCategory(BuiltInCategory.OST_Walls)   # ← change category
                 .WhereElementIsNotElementType()
                 .ToElements())

    # ── Extract data from each element ──────────────────────────────────────
    results = []
    for el in collector:
        try:
            row = {
                'Id':       el.Id.IntegerValue,
                'Name':     el.Name if hasattr(el, 'Name') else 'N/A',
                'Category': el.Category.Name if el.Category else 'N/A',
                # Add more fields here, e.g.:
                # 'Mark': el.get_Parameter(BuiltInParameter.ALL_MODEL_MARK).AsString(),
                # 'Type': doc.GetElement(el.GetTypeId()).Name,
            }
            results.append(row)
        except Exception as row_e:
            results.append({'Id': el.Id.IntegerValue, 'Error': str(row_e)})

    OUT = results

except Exception as e:
    import traceback
    OUT = 'ERROR: ' + str(e) + '\n' + traceback.format_exc()
```

## Common collector patterns

**Filter by parameter value:**
```python
matching = [el for el in collector
            if el.LookupParameter('Fire Rating')
            and el.LookupParameter('Fire Rating').AsString() == '2HR']
```

**Filter elements on a specific level:**
```python
from Autodesk.Revit.DB import ElementLevelFilter, ElementId
level_id = ElementId(12345)  # get this from LookupTable or Dynamo input
level_filter = ElementLevelFilter(level_id)
collector.WherePasses(level_filter)
```

**Get element type name:**
```python
type_id = el.GetTypeId()
if type_id != ElementId.InvalidElementId:
    type_name = doc.GetElement(type_id).Name
```

**Common BuiltInCategory values:**
```
OST_Walls       OST_Floors      OST_Roofs       OST_Ceilings
OST_Doors       OST_Windows     OST_Rooms       OST_Stairs
OST_Sheets      OST_Views       OST_Levels      OST_Grids
OST_StructuralColumns           OST_StructuralFraming
OST_DuctCurves  OST_PipeCurves  OST_Families
```

## Exporting query results to CSV

If the user wants to save the results:

```python
import csv
output_path = r'C:\Reports\query_output.csv'
if results:
    with open(output_path, 'w') as f:
        writer = csv.DictWriter(f, fieldnames=results[0].keys())
        writer.writeheader()
        writer.writerows(results)
    OUT = ['Saved ' + str(len(results)) + ' rows to: ' + output_path, results]
```

Always mention the output path in customization tips so the user knows where to edit it.
