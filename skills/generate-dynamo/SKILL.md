---
name: generate-dynamo
description: >
  Generate a complete, ready-to-run Dynamo Python script for Autodesk Revit from plain language.
  Use this skill whenever a Revit user, architect, or BIM manager asks to automate anything in Revit —
  renaming elements, filtering by parameters, exporting data, modifying families, checking model
  health, batch-editing anything, or any other scripting task. Trigger on phrases like "write a
  Dynamo script", "create a script for Revit", "automate in Revit", "Dynamo Python", or whenever
  someone describes a repetitive Revit workflow that would benefit from automation — even if they
  don't say the word "script". Proactively offer to generate a script when you detect a tedious
  manual workflow.
---

# Generate Dynamo Script

Generate a complete, ready-to-use Dynamo Python script for Revit based on:

**"$ARGUMENTS"**

## Before writing code

Make sure you understand:
1. **What elements** are involved? (walls, rooms, doors, families, sheets, views, etc.)
2. **What action** needs to happen? (read, rename, filter, export, create, modify, etc.)
3. **Which parameters** are involved? Ask for exact names if mentioned — they're case-sensitive.
4. **What's the desired output?** (modified model, printed list, CSV file, count, etc.)

If the request is ambiguous, ask **one focused question** before proceeding. Architects are busy.

## Output format

Always structure the response in this exact order:

### 1. What this script does (2–4 sentences)
Plain English. No jargon. If it modifies the model, say so clearly. If it's read-only, reassure them.

### 2. How to use it
Numbered steps:
1. Open Dynamo (Manage → Dynamo)
2. Create a new graph, place a **Python Script** node
3. Paste the code into the node
4. Connect any Dynamo inputs (explain what each `IN[0]`, `IN[1]` should be)
5. Click **Run**

### 3. The script
Complete, runnable code in a `python` code block. See writing guidelines below.

### 4. Customization tips (optional)
Call out obvious tweaks: parameter names, output paths, category filters, date formats, etc.

---

## Script writing guidelines

### Default: IronPython 2.7 (Dynamo 2.x)
Most firms are on Dynamo 2.x. Switch to CPython 3 only if the user asks for Dynamo 3+.
- IronPython: use `clr`, `TransactionManager`, no f-strings, `DisplayUnitType` for unit conversion
- CPython 3: f-strings OK, use `UnitTypeId` instead of `DisplayUnitType`

### Standard boilerplate — always include

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager

clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

clr.AddReference('RevitAPIUI')
from Autodesk.Revit.UI import *

doc  = DocumentManager.Instance.CurrentDBDocument
uidoc = DocumentManager.Instance.CurrentUIApplication.ActiveUIDocument
```

### Transactions — required for any model modification

```python
TransactionManager.Instance.EnsureInTransaction(doc)
try:
    # ... your modifications ...
    TransactionManager.Instance.TransactionTaskDone()
except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    raise e
```

Read-only scripts (querying / exporting) don't need a transaction.

### Accessing elements

```python
# All elements of a category
collector = FilteredElementCollector(doc) \
    .OfCategory(BuiltInCategory.OST_Rooms) \
    .WhereElementIsNotElementType() \
    .ToElements()

# Common BuiltInCategory values:
# Rooms:OST_Rooms  Walls:OST_Walls  Doors:OST_Doors  Windows:OST_Windows
# Floors:OST_Floors  Ceilings:OST_Ceilings  Stairs:OST_Stairs
# Sheets:OST_Sheets  Views:OST_Views  Levels:OST_Levels  Grids:OST_Grids
# StructuralColumns:OST_StructuralColumns  Families:OST_Families
```

### Reading and writing parameters

```python
# Read
param = element.LookupParameter("Parameter Name")  # user-defined
param = element.get_Parameter(BuiltInParameter.ROOM_NUMBER)  # built-in
if param:
    value = param.AsString()    # text
    value = param.AsDouble()    # number (internal feet — convert!)
    value = param.AsInteger()   # integer / Yes-No (1/0)

# Write
param = element.LookupParameter("Parameter Name")
if param and not param.IsReadOnly:
    param.Set("new value")   # text
    param.Set(3.28084 * m)   # length in feet (Revit internal unit)
```

### Unit conversion (IronPython)
Revit stores lengths in **decimal feet** internally.

```python
from Autodesk.Revit.DB import UnitUtils, DisplayUnitType
feet = UnitUtils.Convert(value_mm, DisplayUnitType.DUT_MILLIMETERS,
                          DisplayUnitType.DUT_DECIMAL_FEET)
mm   = UnitUtils.Convert(value_ft, DisplayUnitType.DUT_DECIMAL_FEET,
                          DisplayUnitType.DUT_MILLIMETERS)
# Simple shorthand: multiply feet × 304.8 to get mm
```

### Error handling — always wrap the main logic

```python
try:
    # main logic
    OUT = result
except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

### Always end with OUT

```python
OUT = result  # count, list, message, or data — give the user confirmation it worked
```

### Comment generously
The person running this may not be a programmer. Explain what each block does and why.

---

## Quality checklist (run before every output)
- [ ] Correct imports for the Dynamo version?
- [ ] Transaction wrapper if the model is modified?
- [ ] Parameter names flagged as case-sensitive if uncertain?
- [ ] try/except with ForceCloseTransaction in the except?
- [ ] OUT = ... at the end?
- [ ] Comments on each key block?
- [ ] Unit conversions handled (feet ↔ mm)?
- [ ] Edge cases: null level, missing parameter, no elements found, exterior doors?
