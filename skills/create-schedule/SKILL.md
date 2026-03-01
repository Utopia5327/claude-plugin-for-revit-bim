---
name: create-schedule
description: >
  Create, configure, or automate Revit schedules, quantity takeoffs, and material takeoffs
  using Dynamo Python. Use this skill when creating element schedules programmatically,
  configuring schedule fields and filters via script, automating schedule creation from a
  list, or producing quantity reports. Trigger on phrases like "create a schedule", "make a
  schedule", "add schedule fields", "automate schedules", "quantity takeoff", "door schedule",
  "wall schedule", "room schedule", or any request to generate or configure a Revit schedule
  via Dynamo. Also offer this skill when users want to batch-create multiple schedules at once.
---

# Create Revit Schedule

Create a Revit schedule for:

**"$ARGUMENTS"**

## Before scripting

Clarify:
- **Which category?** (Doors, Walls, Rooms, Windows, etc.)
- **Which fields** should appear in the schedule? (Mark, Type, Width, Height, Area, etc.)
- **Should it be placed on a sheet?** If so, which sheet number?
- **Any filters?** (e.g. only doors on Level 2, only walls with Fire Rating = 2HR)

## Schedule creation script

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Configuration — edit these ───────────────────────────────────────────────
SCHEDULE_NAME = 'BIM Manager - Door Schedule'
CATEGORY      = BuiltInCategory.OST_Doors   # ← change category as needed

# Built-in parameters to add as schedule fields
# (comment out the ones you don't need, add others)
DESIRED_PARAMS = [
    BuiltInParameter.ALL_MODEL_MARK,
    BuiltInParameter.ALL_MODEL_TYPE_NAME,
    BuiltInParameter.DOOR_WIDTH,
    BuiltInParameter.DOOR_HEIGHT,
    BuiltInParameter.ALL_MODEL_DESCRIPTION,
    BuiltInParameter.ALL_MODEL_MANUFACTURER,
]

TransactionManager.Instance.EnsureInTransaction(doc)
try:
    # ── Create the schedule ──────────────────────────────────────────────────
    category_id = ElementId(CATEGORY)
    schedule    = ViewSchedule.CreateSchedule(doc, category_id)
    schedule.Name = SCHEDULE_NAME

    # ── Get all schedulable fields ───────────────────────────────────────────
    all_fields = list(schedule.Definition.GetSchedulableFields())

    # ── Add desired parameter fields ─────────────────────────────────────────
    desired_ids = set(ElementId(p) for p in DESIRED_PARAMS)
    added = 0
    for sf in all_fields:
        if sf.ParameterId in desired_ids:
            schedule.Definition.AddField(sf)
            added += 1

    # Fallback: if none matched, add the first 6 available fields
    if added == 0:
        for sf in all_fields[:6]:
            schedule.Definition.AddField(sf)

    # ── Add sorting by the first field ──────────────────────────────────────
    if all_fields:
        sort_field = ScheduleSortGroupField(all_fields[0].FieldId)
        schedule.Definition.AddSortGroupField(sort_field)

    TransactionManager.Instance.TransactionTaskDone()
    OUT = 'Schedule created: ' + schedule.Name + ' (Id: ' + str(schedule.Id.IntegerValue) + ')'

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = 'ERROR: ' + str(e) + '\n' + traceback.format_exc()
```

## Adding a filter to the schedule

After creating the schedule, add a filter to show only elements matching a condition:

```python
# Example: show only doors wider than 900mm (≈ 2.95 ft)
min_width_ft = 900 / 304.8

# Get the Width field
width_field = next((f for f in schedule.Definition.GetFields()
                    if f.GetName() == 'Width'), None)
if width_field:
    rule = ScheduleFilterValueRule(
        width_field.FieldId,
        ScheduleFilterType.GreaterThanOrEqual,
        min_width_ft
    )
    flt = ScheduleFilter(rule)
    schedule.Definition.AddFilter(flt)
```

## Placing the schedule on a sheet

```python
# IN[0] = sheet number string (e.g. "A-001")
sheet_number = IN[0]
sheet = next((v for v in FilteredElementCollector(doc)
              .OfClass(ViewSheet).ToElements()
              if v.SheetNumber == sheet_number), None)

if sheet:
    TransactionManager.Instance.EnsureInTransaction(doc)
    # Place at a point on the sheet (adjust coordinates as needed)
    placement_point = XYZ(0.1, 0.1, 0)
    ScheduleSheetInstance.Create(doc, sheet.Id, schedule.Id, placement_point)
    TransactionManager.Instance.TransactionTaskDone()
    OUT = 'Schedule placed on sheet: ' + sheet_number
else:
    OUT = 'Sheet not found: ' + sheet_number
```

## Batch-create multiple schedules

If the user wants several schedules at once (one per level, or one per discipline):

```python
# IN[0] = list of [schedule_name, category_bic_name] pairs
# Example: [["Level 1 Doors", "OST_Doors"], ["Level 1 Rooms", "OST_Rooms"]]
schedule_defs = IN[0]
created = []

TransactionManager.Instance.EnsureInTransaction(doc)
for name, bic_name in schedule_defs:
    bic   = getattr(BuiltInCategory, bic_name)
    sched = ViewSchedule.CreateSchedule(doc, ElementId(bic))
    sched.Name = name
    # Add first 5 fields automatically
    for sf in list(sched.Definition.GetSchedulableFields())[:5]:
        sched.Definition.AddField(sf)
    created.append(name)
TransactionManager.Instance.TransactionTaskDone()
OUT = 'Created: ' + str(created)
```
