---
name: batch-rename-elements
description: >
  Batch rename Revit elements across any category using prefixes, suffixes, numbering sequences,
  or parameter-driven naming rules. Use this skill when a user needs to rename rooms, sheets,
  views, doors, levels, grids, or any other elements en masse — following a naming convention,
  BEP standard, or project numbering scheme. Trigger on phrases like "rename elements", "bulk
  rename", "batch rename", "add prefix", "add suffix", "renumber rooms", "rename views",
  "rename sheets", "apply naming convention", "fix names", or any request to systematically
  change element names at scale.
---

# Batch Rename Elements

Rename Revit elements in bulk for:

**"$ARGUMENTS"**

## Before writing code

Ask (one question only if ambiguous):
1. **Which category?** Rooms, Views, Sheets, Doors, Levels, Grids, or another?
2. **What's the naming pattern?** Prefix + number? Parameter-driven? Find & replace? Sequential?
3. **What parameter holds the name?** Usually `Name` for rooms/views, `Sheet Number`+`Sheet Name` for sheets.

Default: produce a complete, ready-to-run Dynamo script.

---

## Standard rename patterns

### Pattern A — Add prefix/suffix to all elements in a category

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ────────────────────────────────────────────────────────────
CATEGORY    = BuiltInCategory.OST_Rooms   # change to target category
PARAM_NAME  = "Name"                       # parameter that holds the name
PREFIX      = "A-"                         # set "" to skip
SUFFIX      = ""                           # set "" to skip
# ─────────────────────────────────────────────────────────────────────────────

try:
    elements = (FilteredElementCollector(doc)
                .OfCategory(CATEGORY)
                .WhereElementIsNotElementType()
                .ToElements())

    renamed = []
    skipped = []

    TransactionManager.Instance.EnsureInTransaction(doc)

    for el in elements:
        param = el.LookupParameter(PARAM_NAME)
        if param and not param.IsReadOnly:
            old_name = param.AsString() or ""
            # Avoid double-prefixing on re-runs
            if not old_name.startswith(PREFIX):
                new_name = PREFIX + old_name + SUFFIX
                param.Set(new_name)
                renamed.append(old_name + " → " + new_name)
            else:
                skipped.append(old_name + " (already prefixed)")

    TransactionManager.Instance.TransactionTaskDone()

    OUT = {
        "Renamed": renamed,
        "Skipped (already prefixed)": skipped,
        "Total renamed": len(renamed)
    }

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

### Pattern B — Sequential numbering (e.g. Room 001, Room 002 …)

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ────────────────────────────────────────────────────────────
CATEGORY     = BuiltInCategory.OST_Rooms
PARAM_NAME   = "Number"          # parameter to write to
BASE_PREFIX  = "R"               # e.g. "R" → R001, R002 …
START_NUMBER = 1
ZERO_PAD     = 3                 # digits to pad: 3 → 001
SORT_PARAM   = "Name"            # sort elements by this before numbering
# ─────────────────────────────────────────────────────────────────────────────

try:
    elements = list(FilteredElementCollector(doc)
                    .OfCategory(CATEGORY)
                    .WhereElementIsNotElementType()
                    .ToElements())

    # Sort alphabetically by name for consistent numbering
    def get_sort_key(el):
        p = el.LookupParameter(SORT_PARAM)
        return p.AsString() if p else ""

    elements.sort(key=get_sort_key)

    results = []
    TransactionManager.Instance.EnsureInTransaction(doc)

    for i, el in enumerate(elements):
        param = el.LookupParameter(PARAM_NAME)
        if param and not param.IsReadOnly:
            number = BASE_PREFIX + str(START_NUMBER + i).zfill(ZERO_PAD)
            old_val = param.AsString() or ""
            param.Set(number)
            results.append(old_val + " → " + number)

    TransactionManager.Instance.TransactionTaskDone()
    OUT = {"Renumbered": results, "Total": len(results)}

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

### Pattern C — Find & Replace in names

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ────────────────────────────────────────────────────────────
CATEGORY     = BuiltInCategory.OST_Views
PARAM_NAME   = "Name"
FIND         = "Copy of "         # text to find
REPLACE_WITH = ""                 # replace with (empty = remove it)
CASE_SENS    = False              # True = case-sensitive match
# ─────────────────────────────────────────────────────────────────────────────

try:
    elements = (FilteredElementCollector(doc)
                .OfCategory(CATEGORY)
                .WhereElementIsNotElementType()
                .ToElements())

    results = []
    TransactionManager.Instance.EnsureInTransaction(doc)

    for el in elements:
        param = el.LookupParameter(PARAM_NAME)
        if param and not param.IsReadOnly:
            old_name = param.AsString() or ""
            check_name = old_name if CASE_SENS else old_name.lower()
            check_find = FIND if CASE_SENS else FIND.lower()
            if check_find in check_name:
                if CASE_SENS:
                    new_name = old_name.replace(FIND, REPLACE_WITH)
                else:
                    import re
                    new_name = re.sub(re.escape(FIND), REPLACE_WITH, old_name, flags=re.IGNORECASE)
                param.Set(new_name)
                results.append('"' + old_name + '" → "' + new_name + '"')

    TransactionManager.Instance.TransactionTaskDone()
    OUT = {"Renamed": results, "Count": len(results)}

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Category reference

| Target         | BuiltInCategory              | Name param       |
|----------------|------------------------------|------------------|
| Rooms          | `OST_Rooms`                  | `Name`, `Number` |
| Views          | `OST_Views`                  | `Name`           |
| Sheets         | `OST_Sheets`                 | `Name`, `Sheet Number` |
| Doors          | `OST_Doors`                  | `Mark`           |
| Windows        | `OST_Windows`                | `Mark`           |
| Levels         | `OST_Levels`                 | `Name`           |
| Grids          | `OST_Grids`                  | `Name`           |
| Areas          | `OST_Areas`                  | `Name`, `Number` |

## Safety notes

- Always warn the user this **modifies the model** — recommend testing on a detached copy first.
- The prefix-check (`startswith`) prevents double-prefixing on re-runs.
- For sheets, `Sheet Number` and `Name` are separate parameters — handle both if needed.

## Output

Tell the user:
- How many elements were renamed
- A sample of the before/after pairs
- Which elements were skipped and why
