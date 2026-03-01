---
name: audit-model
description: >
  Audit and analyze Revit model health, quality, and BIM standards compliance.
  Use this skill when checking model file size, warning counts, element counts by category,
  redundant or in-place families, unused views, workset structure, model warnings, or any
  model performance or quality-control question. Trigger on phrases like "audit my model",
  "check model health", "how many warnings", "is my model clean", "model performance",
  "BIM QC", "quality check", or whenever someone wants a summary of what's in their Revit model.
  Proactively offer an audit when users mention slow model performance or submission deadlines.
---

# Revit Model Audit

Audit the Revit model for:

**"$ARGUMENTS"**

## Approach

Before scripting, clarify:
- Is this a **comprehensive health report** or a **targeted audit** (e.g. just warnings, just families)?
- Does the user want a **Dynamo script** to run inside Revit, or just guidance on what to look for?

Default to producing a complete Dynamo audit script unless the user asks otherwise.

## Comprehensive model health script

This read-only script collects a full health report — no model modifications, safe to run on any project.

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument
report = {}

try:
    # ── 1. Project information ──────────────────────────────────────────────
    pi = doc.ProjectInformation
    report['Project Name']   = pi.Name
    report['Project Number'] = pi.Number
    report['Client']         = pi.ClientName
    report['Revit Version']  = doc.Application.VersionName

    # ── 2. Element counts by category ──────────────────────────────────────
    cats = [
        ('Walls',            BuiltInCategory.OST_Walls),
        ('Floors',           BuiltInCategory.OST_Floors),
        ('Roofs',            BuiltInCategory.OST_Roofs),
        ('Ceilings',         BuiltInCategory.OST_Ceilings),
        ('Doors',            BuiltInCategory.OST_Doors),
        ('Windows',          BuiltInCategory.OST_Windows),
        ('Rooms',            BuiltInCategory.OST_Rooms),
        ('Stairs',           BuiltInCategory.OST_Stairs),
        ('Struct. Columns',  BuiltInCategory.OST_StructuralColumns),
        ('Struct. Framing',  BuiltInCategory.OST_StructuralFraming),
        ('MEP Ducts',        BuiltInCategory.OST_DuctCurves),
        ('MEP Pipes',        BuiltInCategory.OST_PipeCurves),
        ('Levels',           BuiltInCategory.OST_Levels),
        ('Grids',            BuiltInCategory.OST_Grids),
    ]
    counts = {}
    for name, cat in cats:
        n = (FilteredElementCollector(doc)
             .OfCategory(cat)
             .WhereElementIsNotElementType()
             .GetElementCount())
        counts[name] = n
    report['Element Counts'] = counts

    # ── 3. Families ─────────────────────────────────────────────────────────
    families = list(FilteredElementCollector(doc).OfClass(Family).ToElements())
    report['Total Families']    = len(families)
    report['In-Place Families'] = sum(1 for f in families if f.IsInPlace)

    # ── 4. Views and sheets ─────────────────────────────────────────────────
    all_views = list(FilteredElementCollector(doc).OfClass(View).ToElements())
    sheets    = [v for v in all_views if isinstance(v, ViewSheet)]
    not_on_sheet = [v for v in all_views
                    if not isinstance(v, ViewSheet)
                    and not v.IsTemplate
                    and v.CanBePrinted]
    report['Total Views']       = len(all_views)
    report['Total Sheets']      = len(sheets)
    report['Printable Views']   = len(not_on_sheet)

    # ── 5. Model warnings ───────────────────────────────────────────────────
    warnings = list(doc.GetWarnings())
    report['Model Warnings'] = len(warnings)

    # ── 6. Worksets ─────────────────────────────────────────────────────────
    report['Is Workshared'] = doc.IsWorkshared
    if doc.IsWorkshared:
        wt = (FilteredWorksetCollector(doc)
              .OfKind(WorksetKind.UserWorkset)
              .ToWorksets())
        report['User Worksets'] = len(list(wt))

    OUT = report

except Exception as e:
    import traceback
    OUT = 'ERROR: ' + str(e) + '\n' + traceback.format_exc()
```

## Interpreting results

Guide the user on what the numbers mean:

- **Warnings > 50**: investigate — common culprits are duplicate rooms, overlapping elements, unresolved references
- **In-place families > 10**: performance risk; recommend converting to loadable families
- **Warnings > 500**: flag as a model health issue requiring cleanup before submission
- **Printable views not on sheets**: opportunity to clean up unused views

## Targeted audit variants

If the user wants something specific, tailor the script:

- **Warnings only**: `doc.GetWarnings()` — iterate and group by `GetDescriptionText()`
- **Unused families**: compare `FilteredElementCollector(doc).OfClass(Family)` against `FilteredElementCollector(doc).OfClass(FamilyInstance)`
- **Linked files**: `FilteredElementCollector(doc).OfClass(RevitLinkInstance)`
- **Import CAD links**: `FilteredElementCollector(doc).OfClass(ImportInstance)`

## Output

Always recommend the user export the report to CSV or paste it into a spreadsheet for distribution:

```python
import csv, os
output_path = r'C:\Reports\model_audit.csv'
with open(output_path, 'w') as f:
    writer = csv.writer(f)
    for k, v in report.items():
        if isinstance(v, dict):
            for sub_k, sub_v in v.items():
                writer.writerow([k, sub_k, sub_v])
        else:
            writer.writerow([k, '', v])
OUT = 'Report saved to: ' + output_path
```
