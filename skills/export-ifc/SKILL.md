---
name: export-ifc
description: >
  Export a Revit model to IFC format with proper settings, class mapping, and property sets
  using Dynamo Python. Use this skill when exporting to IFC 2x3 or IFC 4, configuring IFC
  export options, preparing a model for open BIM workflows, or generating IFC files for
  coordination, submission, or client delivery. Trigger on phrases like "export IFC", "IFC
  export", "open BIM", "IFC 2x3", "IFC 4", "BIM submission", "COBie", "IFC coordination",
  or when users mention needing to share the model with contractors, engineers, or clients
  using different software. Also offer this when users mention planning or building permit
  submissions that require open BIM formats.
---

# Export Revit Model to IFC

Configure and export IFC for:

**"$ARGUMENTS"**

## Before scripting

Clarify:
- **IFC version?** IFC 2x3 CV2 (most compatible) or IFC 4 (newer, for Revit 2019+)?
- **Whole model or active view only?** (`VisibleElementsOfCurrentView`)
- **Include linked files?** (`ExportLinkedFiles`)
- **Base quantities?** (area, volume, length — usually yes for quantity takeoffs)
- **Where to save?** Get the output folder from user

## Pre-export checklist

Tell the user to verify before running:
- [ ] All rooms/spaces are placed and bounded (unbounded rooms won't export correctly)
- [ ] Elements are properly categorised (generic models won't map to IFC classes)
- [ ] Project location and coordinates are set (`Manage → Location`)
- [ ] Levels are named consistently
- [ ] Linked files are loaded if `ExportLinkedFiles = True`

## IFC export script

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

# ── Inputs from Dynamo ───────────────────────────────────────────────────────
# IN[0] = export folder path (string), e.g. r"C:\Projects\MyProject\IFC"
# IN[1] = file name (string, no extension), e.g. "MyProject_Architecture_v01"
export_folder   = IN[0]
export_filename = IN[1]

try:
    # ── Configure IFC export options ─────────────────────────────────────────
    ifc_options = IFCExportOptions()

    # IFC version — change to IFCVersion.IFC4 for newer workflows
    ifc_options.FileVersion = IFCVersion.IFC2x3CV2

    # Export settings
    ifc_options.SpaceBoundaryLevel          = 1      # 0=None, 1=1st level, 2=2nd level
    ifc_options.ExportBaseQuantities        = True   # area, volume, length
    ifc_options.WallAndColumnSplitting      = True   # split at levels
    ifc_options.VisibleElementsOfCurrentView = False # False = entire model
    ifc_options.Export2DElements            = False
    ifc_options.ExportLinkedFiles           = False
    ifc_options.ExportSolidModelRep         = False

    # ── Run export ───────────────────────────────────────────────────────────
    doc.Export(export_folder, export_filename, ifc_options)

    full_path = export_folder + '\\' + export_filename + '.ifc'
    OUT = 'IFC exported successfully to: ' + full_path

except Exception as e:
    import traceback
    OUT = 'ERROR: ' + str(e) + '\n' + traceback.format_exc()
```

## IFC class mapping by discipline

Explain to the user how Revit categories map to IFC entities:

| Revit Category        | IFC Entity                  |
|-----------------------|-----------------------------|
| Walls                 | IfcWall / IfcWallStandardCase |
| Floors                | IfcSlab (FloorType)         |
| Roofs                 | IfcRoof                     |
| Columns (Arch)        | IfcColumn                   |
| Structural Columns    | IfcColumn                   |
| Structural Framing    | IfcBeam                     |
| Doors                 | IfcDoor                     |
| Windows               | IfcWindow                   |
| Rooms                 | IfcSpace                    |
| Ducts                 | IfcDuctSegment              |
| Pipes                 | IfcPipeSegment              |
| Generic Models        | IfcBuildingElementProxy     |

If elements appear as `IfcBuildingElementProxy` in the IFC file, they need to be
recategorised in Revit or their IFC export class needs to be overridden via a shared
parameter `IFCExportAs`.

## Post-export validation

Recommend free tools:
- **BIMvision** (free): open IFC file, check hierarchy and properties
- **Solibri Anywhere** (free): validate model health and IFC structure
- **Navisworks Freedom** (free): combine IFC with other disciplines for visual check

## IFC 4 differences

If the user needs IFC 4:
```python
ifc_options.FileVersion = IFCVersion.IFC4
# Also: use IFCVersion.IFC4RV for Reference View (design coordination)
# or IFCVersion.IFC4DTV for Design Transfer View (full geometry)
```

IFC 4 requires Revit 2019 or later.
