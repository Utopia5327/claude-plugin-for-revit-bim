---
name: revit-api-code
description: >
  Generate Revit API code in C# or Python for macros, external commands, add-ins, or pyRevit
  scripts — production-ready, beyond what Dynamo can do. Use this skill when the user needs
  C# Revit macros, IExternalCommand boilerplate, .addin manifest files, pyRevit button scripts,
  or any code that runs outside of Dynamo. Trigger on phrases like "Revit macro", "external
  command", "add-in", "C# Revit", "pyRevit script", "Revit API code", "IExternalCommand",
  "ribbon button", "addin", or when a user asks for Revit automation that needs to be
  deployed as a standalone tool rather than a Dynamo graph. Also offer this when users want
  persistent tools that run without opening Dynamo each time.
---

# Generate Revit API Code

Generate Revit API code for:

**"$ARGUMENTS"**

## Clarify the target

Before writing code, confirm:
1. **C# or Python?**
   - C# → Revit macro (internal, `.cs` file in Revit's SharpDevelop IDE) or add-in (`.dll`)
   - Python → pyRevit script (`.py` file in a pyRevit extension bundle)
2. **Where will it run?**
   - Macro: runs from Manage → Macros, no installation needed
   - External command: compiled `.dll`, registered in `.addin` file
   - pyRevit: drop into extension folder, appears as ribbon button automatically
3. **Revit version?** Affects API availability.

---

## C# External Command (Add-in)

Complete, production-ready boilerplate for an `IExternalCommand`:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Autodesk.Revit.Attributes;
using Autodesk.Revit.DB;
using Autodesk.Revit.UI;

namespace RevitBIMPlugin
{
    [Transaction(TransactionMode.Manual)]
    [Regeneration(RegenerationOption.Manual)]
    public class MyCommand : IExternalCommand
    {
        public Result Execute(
            ExternalCommandData commandData,
            ref string message,
            ElementSet elements)
        {
            UIApplication uiapp = commandData.Application;
            UIDocument    uidoc = uiapp.ActiveUIDocument;
            Document      doc   = uidoc.Document;

            try
            {
                using (Transaction tx = new Transaction(doc, "My BIM Operation"))
                {
                    tx.Start();

                    // ── Your code here ────────────────────────────────
                    // Example: count all walls
                    var walls = new FilteredElementCollector(doc)
                        .OfCategory(BuiltInCategory.OST_Walls)
                        .WhereElementIsNotElementType()
                        .ToElements();

                    tx.Commit();

                    TaskDialog.Show("BIM Manager",
                        $"Operation completed. Found {walls.Count} walls.");
                }

                return Result.Succeeded;
            }
            catch (Exception ex)
            {
                message = ex.Message;
                return Result.Failed;
            }
        }
    }
}
```

**Corresponding `.addin` manifest** (save as `MyPlugin.addin` in `%AppData%\Autodesk\Revit\Addins\20XX\`):
```xml
<?xml version="1.0" encoding="utf-8"?>
<RevitAddIns>
  <AddIn Type="Command">
    <Name>My BIM Command</Name>
    <Assembly>C:\Path\To\MyPlugin.dll</Assembly>
    <FullClassName>RevitBIMPlugin.MyCommand</FullClassName>
    <ClientId>YOUR-GUID-HERE</ClientId>
    <VendorId>YOURINITIALS</VendorId>
    <VendorDescription>Your Company</VendorDescription>
  </AddIn>
</RevitAddIns>
```

---

## C# Revit Macro

For quick scripts run directly from Revit without compiling a DLL:

```csharp
// In Revit: Manage → Macros → Module → add this method
public void MyMacro()
{
    Document doc = this.ActiveUIDocument.Document;

    using (Transaction tx = new Transaction(doc, "Macro Operation"))
    {
        tx.Start();

        // ── Your code here ────────────────────────────────────────
        var rooms = new FilteredElementCollector(doc)
            .OfCategory(BuiltInCategory.OST_Rooms)
            .WhereElementIsNotElementType()
            .ToElements();

        TaskDialog.Show("Macro", $"Found {rooms.Count} rooms.");

        tx.Commit();
    }
}
```

---

## pyRevit Script (Python)

For ribbon button scripts using the pyRevit framework:

```python
# -*- coding: utf-8 -*-
"""BIM Manager — pyRevit Script.
tooltip: Brief description shown on hover
"""
from pyrevit import revit, DB, UI, script, forms

doc   = revit.doc
uidoc = revit.uidoc
output = script.get_output()

# ── Your code here ───────────────────────────────────────────────────────────
# Read elements
rooms = list(DB.FilteredElementCollector(doc)
             .OfCategory(DB.BuiltInCategory.OST_Rooms)
             .WhereElementIsNotElementType()
             .ToElements())

# Modify with transaction
with revit.Transaction('BIM Operation'):
    pass  # ← your modifications

# Output to pyRevit's output panel (supports HTML/markdown)
output.print_md('## Done!')
output.print_table(
    table_data=[[r.Number, r.get_Parameter(DB.BuiltInParameter.ROOM_NAME).AsString()]
                for r in rooms],
    columns=['Number', 'Name']
)
```

**pyRevit extension structure:**
```
MyExtension.extension/
├── MyPanel.panel/
│   └── MyButton.pushbutton/
│       ├── script.py        ← your code above
│       └── icon.png         ← 32×32 PNG
└── extension.json
```

---

## Useful API snippets

**Show a TaskDialog with user options:**
```csharp
TaskDialogResult result = TaskDialog.Show("Confirm",
    "This will modify all wall types. Proceed?",
    TaskDialogCommonButtons.Yes | TaskDialogCommonButtons.No);
if (result != TaskDialogResult.Yes) return Result.Cancelled;
```

**Select elements in the UI:**
```csharp
IList<Reference> refs = uidoc.Selection.PickObjects(
    ObjectType.Element, "Select elements to process");
var selected = refs.Select(r => doc.GetElement(r.ElementId)).ToList();
```

**Print to Output window (pyRevit):**
```python
output.print_md('**Found** {} walls'.format(len(walls)))
output.print_table(data, columns=['ID', 'Type', 'Length'])
```
