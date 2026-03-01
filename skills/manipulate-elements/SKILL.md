---
name: manipulate-elements
description: >
  Move, rotate, copy, delete, modify types, or batch-edit Revit model elements using
  Dynamo Python. Use this skill when performing actions and geometric transformations on
  elements — translating elements, changing their type, batch-editing families, swapping
  types, mirroring, rotating, or any operation that modifies element geometry or identity.
  Trigger on phrases like "move elements", "rotate", "copy elements", "change type",
  "batch edit", "swap family", "modify elements", "transform", "mirror elements", or any
  request to programmatically manipulate existing Revit elements. Always warn the user
  to test on a copy of the model first before running destructive operations.
---

# Manipulate Revit Elements

Perform the following action on Revit elements:

**"$ARGUMENTS"**

## Before scripting

Important: **always warn the user** — this type of script modifies the model. Recommend:
1. Save a local backup first (Save As → make a copy)
2. Test on a small set of elements before running on the whole model
3. Ctrl+Z works in Revit after Dynamo runs — but only if the transaction closed cleanly

Clarify:
- **Which elements?** Collect by category, or receive from an upstream Dynamo node?
- **What transformation?** Move (vector), rotate (axis + angle), change type (new type ID), etc.
- **How are elements passed in?** As Dynamo-wrapped objects (from nodes) or native Revit elements?

## Element modification template

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

# Unwrap Dynamo-wrapped elements to native Revit elements
clr.AddReference('RevitNodes')
import Revit
clr.ImportExtensions(Revit.Elements)

doc = DocumentManager.Instance.CurrentDBDocument

# IN[0] = list of elements from upstream Dynamo node
elements = IN[0] if isinstance(IN[0], list) else [IN[0]]

modified = []
errors   = []

TransactionManager.Instance.EnsureInTransaction(doc)
try:
    for el in elements:
        # Unwrap if element came from a Dynamo node
        revit_el = el.InternalElement if hasattr(el, 'InternalElement') else el

        # ── Your modification here ───────────────────────────────────────────
        # Example A: Move element 1m in the X direction
        # translation = XYZ(3.28084, 0, 0)  # 1 metre in X (Revit uses feet)
        # ElementTransformUtils.MoveElement(doc, revit_el.Id, translation)

        # Example B: Change element type
        # new_type_id = ElementId(123456)  # get from a Type Collector node
        # revit_el.ChangeTypeId(new_type_id)

        # Example C: Rotate 90° around element's Z axis
        # from Autodesk.Revit.DB import Line
        # origin = revit_el.Location.Point
        # axis   = Line.CreateBound(origin, origin + XYZ.BasisZ)
        # import math
        # ElementTransformUtils.RotateElement(doc, revit_el.Id, axis, math.pi / 2)

        modified.append(revit_el.Id.IntegerValue)

    TransactionManager.Instance.TransactionTaskDone()

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    errors.append(str(e))

OUT = [modified, errors]
```

## Common transformation patterns

**Move all elements in a category by a vector:**
```python
# Shift everything 500mm in Z (e.g. raise a floor level)
z_shift_ft = 500 / 304.8  # convert mm to feet
vector = XYZ(0, 0, z_shift_ft)

collector = (FilteredElementCollector(doc)
             .OfCategory(BuiltInCategory.OST_Floors)
             .WhereElementIsNotElementType()
             .ToElements())

TransactionManager.Instance.EnsureInTransaction(doc)
for el in collector:
    ElementTransformUtils.MoveElement(doc, el.Id, vector)
TransactionManager.Instance.TransactionTaskDone()
OUT = 'Moved ' + str(len(list(collector))) + ' floors by 500mm'
```

**Batch change family type:**
```python
# IN[0] = list of elements
# IN[1] = new type name (string)
# IN[2] = category (e.g. "OST_Doors")
elements   = IN[0]
type_name  = IN[1]
bic        = getattr(BuiltInCategory, IN[2])

new_type = next((t for t in FilteredElementCollector(doc)
                 .OfCategory(bic)
                 .WhereElementIsElementType().ToElements()
                 if t.Name == type_name), None)

if not new_type:
    OUT = 'Type not found: ' + type_name
else:
    TransactionManager.Instance.EnsureInTransaction(doc)
    count = 0
    for el in elements:
        revit_el = el.InternalElement if hasattr(el, 'InternalElement') else el
        revit_el.ChangeTypeId(new_type.Id)
        count += 1
    TransactionManager.Instance.TransactionTaskDone()
    OUT = 'Changed ' + str(count) + ' elements to type: ' + type_name
```

**Mirror elements about a vertical plane:**
```python
# Mirror about the X-axis through the model origin
mirror_plane = Plane.CreateByNormalAndOrigin(XYZ.BasisX, XYZ.Zero)
ids_to_mirror = [el.Id for el in elements]

TransactionManager.Instance.EnsureInTransaction(doc)
ElementTransformUtils.MirrorElements(doc, ids_to_mirror, mirror_plane, True)
TransactionManager.Instance.TransactionTaskDone()
```

## Unit conversion reminder

Revit's internal unit for length is **decimal feet**. Always convert:
- `mm → ft`: divide by 304.8
- `m → ft`: divide by 0.3048
- `ft → mm`: multiply by 304.8

Example: `XYZ(1000 / 304.8, 0, 0)` moves 1000mm in X.
