---
name: manage-sheets
description: >
  Create, organise, and manage Revit drawing sheets and views using Dynamo Python.
  Use this skill when creating sheets from a list (e.g. from Excel), placing views onto
  sheets, applying view templates, renaming or renumbering sheets, automating sheet set
  creation, or reorganising the drawing register. Trigger on phrases like "create sheets",
  "batch create sheets", "add views to sheets", "sheet setup", "drawing register",
  "sheet numbering", "place views", "apply view template", "sheets from Excel", or any
  request involving Revit sheet management. Also offer this skill when users are setting
  up a new project or preparing for a drawing issue.
---

# Manage Revit Sheets & Views

Perform sheet/view operation:

**"$ARGUMENTS"**

## Clarify before scripting

- **Create new sheets** or **modify existing ones?**
- **Where does the sheet list come from?** (Dynamo input, Excel, hardcoded in script?)
- **Which title block** should be used? (Get the first available one, or ask for the name?)
- **Place views on sheets?** If so, which views, at which positions?

## Batch sheet creation script

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Dynamo inputs ────────────────────────────────────────────────────────────
# IN[0] = list of sheet numbers (strings), e.g. ["A-001", "A-002", "A-003"]
# IN[1] = list of sheet names (strings),   e.g. ["Ground Floor Plan", "Roof Plan", "Sections"]
sheet_numbers = IN[0] if isinstance(IN[0], list) else [IN[0]]
sheet_names   = IN[1] if isinstance(IN[1], list) else [IN[1]]

# ── Get the first available title block type ─────────────────────────────────
tb_types = list(FilteredElementCollector(doc)
                .OfCategory(BuiltInCategory.OST_TitleBlocks)
                .WhereElementIsElementType()
                .ToElements())
tb_id = tb_types[0].Id if tb_types else ElementId.InvalidElementId

TransactionManager.Instance.EnsureInTransaction(doc)
created = []
skipped = []

try:
    for number, name in zip(sheet_numbers, sheet_names):
        # Check if sheet number already exists
        existing = [v for v in FilteredElementCollector(doc)
                    .OfClass(ViewSheet).ToElements()
                    if v.SheetNumber == number]
        if existing:
            skipped.append({'Number': number, 'Status': 'Already exists'})
            continue

        sheet = ViewSheet.Create(doc, tb_id)
        sheet.SheetNumber = number
        sheet.Name = name
        created.append({
            'Number': number,
            'Name':   name,
            'Id':     sheet.Id.IntegerValue,
            'Status': 'Created'
        })

    TransactionManager.Instance.TransactionTaskDone()
    OUT = [created, skipped]

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = 'ERROR: ' + str(e) + '\n' + traceback.format_exc()
```

## Place a view onto a sheet

```python
# IN[0] = sheet number (string)
# IN[1] = view name (string)
# IN[2] = placement point [x, y] in sheet coordinates (optional, defaults to centre)
sheet_number = IN[0]
view_name    = IN[1]
point_xy     = IN[2] if len(IN) > 2 else [0.3, 0.3]

sheet = next((v for v in FilteredElementCollector(doc)
              .OfClass(ViewSheet).ToElements()
              if v.SheetNumber == sheet_number), None)

view = next((v for v in FilteredElementCollector(doc)
             .OfClass(View).ToElements()
             if v.Name == view_name
             and not v.IsTemplate), None)

if sheet and view:
    if Viewport.CanAddViewToSheet(doc, sheet.Id, view.Id):
        TransactionManager.Instance.EnsureInTransaction(doc)
        placement = XYZ(point_xy[0], point_xy[1], 0)
        vp = Viewport.Create(doc, sheet.Id, view.Id, placement)
        TransactionManager.Instance.TransactionTaskDone()
        OUT = 'Placed view "' + view_name + '" on sheet ' + sheet_number
    else:
        OUT = 'View is already on a sheet or cannot be placed'
else:
    OUT = 'Sheet or view not found'
```

## Apply a view template to multiple views

```python
# IN[0] = view template name (string)
# IN[1] = list of view names to apply it to
template_name = IN[0]
view_names    = IN[1]

template = next((v for v in FilteredElementCollector(doc)
                 .OfClass(View).ToElements()
                 if v.IsTemplate and v.Name == template_name), None)

if not template:
    OUT = 'Template not found: ' + template_name
else:
    TransactionManager.Instance.EnsureInTransaction(doc)
    applied = []
    for vname in view_names:
        v = next((vw for vw in FilteredElementCollector(doc)
                  .OfClass(View).ToElements()
                  if vw.Name == vname and not vw.IsTemplate), None)
        if v:
            v.ViewTemplateId = template.Id
            applied.append(vname)
    TransactionManager.Instance.TransactionTaskDone()
    OUT = 'Applied template to: ' + str(applied)
```

## Rename / renumber sheets from Excel

If the user has a spreadsheet with new numbers and names:

```python
# IN[0] = 2D list from Excel: [[old_number, new_number, new_name], ...]
data = IN[0]

TransactionManager.Instance.EnsureInTransaction(doc)
results = []
for old_num, new_num, new_name in data:
    sheet = next((v for v in FilteredElementCollector(doc)
                  .OfClass(ViewSheet).ToElements()
                  if v.SheetNumber == str(old_num)), None)
    if sheet:
        sheet.SheetNumber = str(new_num)
        sheet.Name        = str(new_name)
        results.append({'Old': old_num, 'New': new_num, 'Status': 'OK'})
    else:
        results.append({'Old': old_num, 'New': new_num, 'Status': 'NOT FOUND'})
TransactionManager.Instance.TransactionTaskDone()
OUT = results
```

## Customisation tips

- **Title block**: if `tb_types` returns empty, the model has no loaded title block family — load one first from `Insert → Load Family`
- **Sheet coordinates**: Revit sheet coordinates are in feet; `XYZ(0.3, 0.3, 0)` places the view roughly in the lower-left area of a standard sheet
- **Viewport positions**: after placing, use `vp.SetBoxCenter()` to reposition precisely
