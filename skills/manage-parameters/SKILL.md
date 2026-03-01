---
name: manage-parameters
description: >
  Read, write, batch-update, or validate Revit element parameters using Dynamo Python.
  Use this skill when modifying parameter values across many elements, batch-setting data
  from Excel, creating project or shared parameters, mapping parameters between fields,
  checking for missing or empty parameters, or copying values from one parameter to another.
  Trigger on phrases like "set parameter", "update parameters", "batch edit", "fill in",
  "parameter value", "shared parameter", "copy parameter", "parameter missing", or any
  request to read or write Revit element data at scale.
---

# Manage Revit Parameters

Perform parameter operation:

**"$ARGUMENTS"**

## Clarify before scripting

- **Read or write?** Read-only queries don't need a transaction.
- **Which elements?** All elements of a category, or a filtered subset?
- **Which parameter?** Ask for the exact name (case-sensitive) if the user mentions one.
- **What value?** String, number, yes/no, or element reference?

## Read/write parameter script

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

# Unwrap Dynamo-wrapped elements if needed
clr.AddReference('RevitNodes')
import Revit
clr.ImportExtensions(Revit.Elements)

doc = DocumentManager.Instance.CurrentDBDocument

# ── Dynamo inputs ────────────────────────────────────────────────────────────
# IN[0] = list of Revit elements (from a Collector or upstream node)
# IN[1] = parameter name (string)
# IN[2] = new value (string/number) — omit or set to None for read-only
elements   = IN[0] if isinstance(IN[0], list) else [IN[0]]
param_name = IN[1]
new_value  = IN[2] if len(IN) > 2 else None

write_mode = new_value is not None
results    = []

try:
    if write_mode:
        TransactionManager.Instance.EnsureInTransaction(doc)

    for el in elements:
        # Unwrap Dynamo wrapper to native Revit element
        revit_el = el.InternalElement if hasattr(el, 'InternalElement') else el

        param = revit_el.LookupParameter(param_name)

        if param is None:
            results.append({'Id': revit_el.Id.IntegerValue, 'Status': 'NOT FOUND'})
            continue

        if not write_mode:
            # ── Read mode ───────────────────────────────────────────────────
            val = (param.AsValueString()
                   or param.AsString()
                   or str(param.AsDouble()))
            results.append({'Id': revit_el.Id.IntegerValue, param_name: val})
        else:
            # ── Write mode ──────────────────────────────────────────────────
            if param.IsReadOnly:
                results.append({'Id': revit_el.Id.IntegerValue, 'Status': 'READ ONLY'})
                continue

            st = param.StorageType
            if   st == StorageType.String:  param.Set(str(new_value))
            elif st == StorageType.Double:  param.Set(float(new_value))
            elif st == StorageType.Integer: param.Set(int(new_value))
            else:
                results.append({'Id': revit_el.Id.IntegerValue,
                                'Status': 'UNSUPPORTED TYPE: ' + str(st)})
                continue

            results.append({'Id': revit_el.Id.IntegerValue,
                            'Status': 'OK',
                            'New Value': new_value})

    if write_mode:
        TransactionManager.Instance.TransactionTaskDone()

    OUT = results

except Exception as e:
    if write_mode:
        TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = 'ERROR: ' + str(e) + '\n' + traceback.format_exc()
```

## Common parameter patterns

**Check for empty parameters (compliance check):**
```python
missing = []
for el in collector:
    p = el.LookupParameter('Description')
    if not p or not p.AsString() or p.AsString().strip() == '':
        missing.append({'Id': el.Id.IntegerValue,
                        'Mark': el.get_Parameter(BuiltInParameter.ALL_MODEL_MARK).AsString()})
OUT = missing
```

**Copy value from one parameter to another:**
```python
TransactionManager.Instance.EnsureInTransaction(doc)
for el in collector:
    source = el.LookupParameter('Old Param Name')
    target = el.LookupParameter('New Param Name')
    if source and target and not target.IsReadOnly:
        target.Set(source.AsString())
TransactionManager.Instance.TransactionTaskDone()
```

**Batch-set from Excel (via Data.ImportExcel node):**
```python
# IN[0] = 2D list from Excel: [[element_id, new_value], ...]
data = IN[0]
id_value_map = {int(row[0]): row[1] for row in data if row[0]}

TransactionManager.Instance.EnsureInTransaction(doc)
for el_id_int, val in id_value_map.items():
    el = doc.GetElement(ElementId(el_id_int))
    if el:
        p = el.LookupParameter('Target Param')
        if p and not p.IsReadOnly:
            p.Set(str(val))
TransactionManager.Instance.TransactionTaskDone()
```

## Customization tips

- Parameter names are **case-sensitive** — double-check against the Revit properties panel
- For built-in parameters use `element.get_Parameter(BuiltInParameter.ROOM_NUMBER)` for better reliability
- If `AsValueString()` returns `None`, try `AsString()` then `AsDouble()` — storage type varies by parameter
