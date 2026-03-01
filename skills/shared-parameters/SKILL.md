---
name: shared-parameters
description: >
  Create, bind, and manage Revit shared parameters using Dynamo Python — load a shared
  parameter file, add new parameter definitions to groups, bind parameters to element
  categories, and verify parameter bindings. Use this skill when a user needs to add
  shared parameters to their project, bind COBie parameters, set up BEP-required data
  fields, add parameters to families, or troubleshoot missing shared parameters. Trigger
  on phrases like "shared parameters", "add parameter", "bind parameter", "shared param
  file", "project parameters", "create parameter", "COBie fields", "parameter binding",
  "add fields to schedule", "parameter group", or any request to add new data fields
  to Revit elements that don't exist yet.
---

# Shared Parameters — Create, Bind & Manage

Manage shared parameters for:

**"$ARGUMENTS"**

## Before writing code

Ask (one question if ambiguous):
1. **Do you have an existing shared parameter file** (`.txt`), or do you need a new one created?
2. **Which parameters** do you want to add, and to **which categories**?
3. **Instance or Type parameter?** (Instance = per-element, Type = per-family type)

---

## Script 1: Inspect what shared parameters are already bound (read-only)

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

try:
    binding_map = doc.ParameterBindings
    it = binding_map.ForwardIterator()
    results = []

    while it.MoveNext():
        defn    = it.Key
        binding = it.Current
        cats    = []
        try:
            for cat in binding.Categories:
                cats.append(cat.Name)
        except Exception:
            pass

        results.append({
            "Name":       defn.Name,
            "Type":       "Instance" if isinstance(binding, InstanceBinding) else "Type",
            "Categories": ", ".join(sorted(cats))
        })

    results.sort(key=lambda x: x["Name"])
    OUT = results if results else "No shared or project parameters found."

except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Script 2: Load shared parameter file and bind parameters to categories

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument
app = doc.Application

# ── Configuration ────────────────────────────────────────────────────────────
SHARED_PARAM_FILE = r"C:\BIM\SharedParameters\ProjectSharedParams.txt"

# Parameters to bind: (group_name_in_file, param_name, is_instance, categories_list)
PARAMS_TO_BIND = [
    ("COBie",    "COBie.Type.AssetType",    False,  [BuiltInCategory.OST_Doors,
                                                     BuiltInCategory.OST_Windows,
                                                     BuiltInCategory.OST_MechanicalEquipment]),
    ("COBie",    "COBie.Component.TagNumber", True, [BuiltInCategory.OST_Doors,
                                                     BuiltInCategory.OST_MechanicalEquipment]),
    ("BIM Data", "BIM_Status",              True,   [BuiltInCategory.OST_Walls,
                                                     BuiltInCategory.OST_Floors,
                                                     BuiltInCategory.OST_Rooms]),
    ("BIM Data", "BIM_ResponsibleParty",    True,   [BuiltInCategory.OST_Rooms]),
]
# ─────────────────────────────────────────────────────────────────────────────

def get_or_open_sp_file(app, path):
    """Set the shared parameter file and return it."""
    app.SharedParametersFilename = path
    return app.OpenSharedParameterFile()

try:
    import os
    if not os.path.exists(SHARED_PARAM_FILE):
        OUT = ("ERROR: Shared parameter file not found: " + SHARED_PARAM_FILE +
               "\nCreate the file first or check the path.")
    else:
        sp_file = get_or_open_sp_file(app, SHARED_PARAM_FILE)
        if not sp_file:
            OUT = "ERROR: Could not open shared parameter file."
        else:
            bound_params = []
            skipped = []

            TransactionManager.Instance.EnsureInTransaction(doc)

            for group_name, param_name, is_instance, cat_list in PARAMS_TO_BIND:

                # Find the group
                group = sp_file.Groups.get_Item(group_name)
                if not group:
                    skipped.append(param_name + " (group '" + group_name + "' not found in file)")
                    continue

                # Find the definition in the group
                defn = group.Definitions.get_Item(param_name)
                if not defn:
                    skipped.append(param_name + " (not found in group '" + group_name + "')")
                    continue

                # Check if already bound
                existing = doc.ParameterBindings.get_Item(defn)
                if existing:
                    skipped.append(param_name + " (already bound)")
                    continue

                # Build category set
                cat_set = app.Create.NewCategorySet()
                for bic in cat_list:
                    cat = doc.Settings.Categories.get_Item(bic)
                    if cat and cat.AllowsBoundParameters:
                        cat_set.Insert(cat)

                # Create binding
                if is_instance:
                    binding = app.Create.NewInstanceBinding(cat_set)
                else:
                    binding = app.Create.NewTypeBinding(cat_set)

                # Insert into document
                group_defn = BuiltInParameterGroup.PG_DATA  # show under "Data" in Properties
                doc.ParameterBindings.Insert(defn, binding, group_defn)
                bound_params.append(param_name)

            TransactionManager.Instance.TransactionTaskDone()
            OUT = {"Bound": bound_params, "Skipped": skipped}

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Script 3: Create a new shared parameter file with definitions

Use this when no shared parameter file exists yet.

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument
app = doc.Application

# ── Configuration ────────────────────────────────────────────────────────────
NEW_SP_FILE = r"C:\BIM\SharedParameters\NewProjectParams.txt"

# (group_name, param_name, param_type)
# ParameterType options: Text, Integer, Number, Length, Area, YesNo, URL, Material, etc.
PARAMS_TO_CREATE = [
    ("BIM Data", "BIM_Status",             ParameterType.Text),
    ("BIM Data", "BIM_ResponsibleParty",   ParameterType.Text),
    ("BIM Data", "BIM_SubmissionDate",     ParameterType.Text),
    ("COBie",    "COBie.Type.AssetType",   ParameterType.Text),
    ("COBie",    "COBie.Component.TagNumber", ParameterType.Text),
]
# ─────────────────────────────────────────────────────────────────────────────

try:
    import os
    os.makedirs(os.path.dirname(NEW_SP_FILE), exist_ok=True)

    # Create new empty shared parameter file
    app.SharedParametersFilename = NEW_SP_FILE
    # Writing a blank file first so Revit can open it
    if not os.path.exists(NEW_SP_FILE):
        with open(NEW_SP_FILE, 'w') as f:
            f.write("# This is a Revit shared parameter file.\n")
            f.write("# Do not edit manually.\n")
            f.write("*META\tVERSION\tMINVERSION\n")
            f.write("META\t2\t1\n")
            f.write("*GROUP\tID\tNAME\n")
            f.write("*PARAM\tGUID\tNAME\tDATATYPE\tDATACATEGORY\tGROUP\tVISIBLE\tDESCRIPTION\tUSERMODIFIABLE\tHIDEWHENNOVALUE\n")

    sp_file = app.OpenSharedParameterFile()

    created = []
    for group_name, param_name, param_type in PARAMS_TO_CREATE:
        # Get or create group
        group = sp_file.Groups.get_Item(group_name)
        if not group:
            group = sp_file.Groups.Create(group_name)

        # Check if param already exists
        existing = group.Definitions.get_Item(param_name)
        if not existing:
            opts = ExternalDefinitionCreationOptions(param_name, param_type)
            opts.Visible = True
            group.Definitions.Create(opts)
            created.append(group_name + " > " + param_name)

    OUT = {
        "File created": NEW_SP_FILE,
        "Parameters created": created,
        "Next step": "Use Script 2 to bind these parameters to Revit categories."
    }

except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Common BuiltInParameterGroup values

| Display name in Properties | BuiltInParameterGroup           |
|----------------------------|---------------------------------|
| Data                       | `PG_DATA`                       |
| Identity Data              | `PG_IDENTITY_DATA`              |
| Other                      | `PG_OTHER`                      |
| Construction               | `PG_CONSTRUCTION`               |
| Mechanical                 | `PG_MECHANICAL`                 |
| Energy Analysis            | `PG_ENERGY_ANALYSIS`            |

## Troubleshooting

- **"AllowsBoundParameters is false"**: Some categories (e.g. Lines, Detail Items) can't have instance parameters bound via the API — use Type binding or a different category.
- **Parameter not appearing in schedules**: Shared parameters appear in schedules; project parameters do not. Make sure you're using a shared parameter file, not `doc.ParameterBindings.Insert` with an `InternalDefinition`.
- **GUID conflicts**: Each shared parameter has a unique GUID. Never duplicate a GUID across files — Revit uses it to match parameters across projects.
