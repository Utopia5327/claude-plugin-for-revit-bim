---
name: analyze-rooms
description: >
  Analyze rooms, spaces, and areas in a Revit model — GIA calculations, space program
  comparison, department breakdowns, unbounded/unplaced room audits, and CSV export.
  Use this skill when a user wants to calculate total floor area, check room areas against
  a brief, export a room data table, find unplaced or unbounded rooms, or produce a GIA
  (Gross Internal Area) report. Trigger on phrases like "room areas", "GIA", "space program",
  "analyze rooms", "unbounded rooms", "area schedule", "room data", "floor area summary",
  "net internal area", or any request involving Revit room/space quantities. Also use when
  someone mentions area validation before planning submission.
---

# Analyze Revit Rooms & Spaces

Perform room/space analysis for:

**"$ARGUMENTS"**

## Room data extraction script

Read-only — no model modifications.

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# Unit conversion constants
SQ_FT_TO_SQ_M    = 0.092903   # Revit stores area in square feet internally
CUBIC_FT_TO_M3   = 0.0283168
FT_TO_M          = 0.3048

try:
    # Collect all placed rooms
    rooms = (FilteredElementCollector(doc)
             .OfCategory(BuiltInCategory.OST_Rooms)
             .WhereElementIsNotElementType()
             .ToElements())

    room_data = []
    unplaced   = []

    for room in rooms:
        area_sqft = room.Area

        if area_sqft <= 0:
            # Unplaced or unbounded room — flag it
            unplaced.append({
                'Id':     room.Id.IntegerValue,
                'Number': room.Number,
                'Name':   room.get_Parameter(BuiltInParameter.ROOM_NAME).AsString() or '',
                'Issue':  'Unplaced or Unbounded'
            })
            continue

        # Pull all useful fields
        dept_param  = room.get_Parameter(BuiltInParameter.ROOM_DEPARTMENT)
        phase_param = room.get_Parameter(BuiltInParameter.ROOM_PHASE)

        data = {
            'Room Number':   room.Number,
            'Room Name':     room.get_Parameter(BuiltInParameter.ROOM_NAME).AsString() or '',
            'Department':    dept_param.AsString()  if dept_param  else '',
            'Level':         room.Level.Name         if room.Level  else 'N/A',
            'Area (m2)':     round(area_sqft * SQ_FT_TO_SQ_M, 2),
            'Area (ft2)':    round(area_sqft, 2),
            'Volume (m3)':   round(room.Volume * CUBIC_FT_TO_M3, 2),
            'Perimeter (m)': round(room.Perimeter * FT_TO_M, 2),
            'Phase':         phase_param.AsValueString() if phase_param else '',
            'Element Id':    room.Id.IntegerValue,
        }
        room_data.append(data)

    # Sort by level, then room number
    room_data.sort(key=lambda x: (x['Level'], x['Room Number']))

    # Calculate totals
    gia_m2 = sum(r['Area (m2)'] for r in room_data)

    OUT = [
        room_data,                             # full data table
        unplaced,                              # rooms to fix
        round(gia_m2, 2),                      # total GIA in m²
        len(room_data),                        # placed room count
        len(unplaced),                         # unplaced count
    ]

except Exception as e:
    import traceback
    OUT = 'ERROR: ' + str(e) + '\n' + traceback.format_exc()
```

## Exporting to CSV

```python
import csv
output_path = r'C:\Reports\room_analysis.csv'
if room_data:
    with open(output_path, 'w') as f:
        writer = csv.DictWriter(f, fieldnames=room_data[0].keys())
        writer.writeheader()
        writer.writerows(room_data)
    print('Saved to: ' + output_path)
```

## Fixing unplaced / unbounded rooms

Guide the user:
1. Open **Room Separation Lines** view — confirm boundary lines form a closed loop
2. In Revit, go to **Architecture → Room** and check the room boundary visibility
3. Unbounded rooms often mean boundary lines are missing or the room is placed outside a boundary
4. Delete truly unneeded rooms (they appear as "Not Placed" in schedules)

## GIA / NIA breakdown by department

If the user wants breakdown by department or level:

```python
from collections import defaultdict
by_dept = defaultdict(float)
by_level = defaultdict(float)
for r in room_data:
    by_dept[r['Department'] or 'Unassigned'] += r['Area (m2)']
    by_level[r['Level']] += r['Area (m2)']
OUT = [dict(by_dept), dict(by_level), round(gia_m2, 2)]
```

## Space program comparison

If the user has a brief (target areas), accept it as a Dynamo input:

```python
# IN[0] = list of [room_name, target_area_m2] pairs from an Excel node
brief = {row[0]: row[1] for row in IN[0]}
comparison = []
for r in room_data:
    target = brief.get(r['Room Name'])
    if target:
        delta = round(r['Area (m2)'] - float(target), 2)
        comparison.append({
            'Room': r['Room Name'],
            'Actual (m2)': r['Area (m2)'],
            'Target (m2)': float(target),
            'Delta (m2)': delta,
            'Status': 'Over' if delta > 0 else 'Under' if delta < 0 else 'OK'
        })
OUT = comparison
```
