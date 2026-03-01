---
name: create-view-filters
description: >
  Create, configure, and apply Revit view filters and graphic overrides using Dynamo Python.
  Use this skill when a user wants to color-code elements by parameter value, create filters
  based on phase, status, type, department, workset, or any custom parameter, apply those
  filters to multiple views at once, or override element visibility by rule. Trigger on
  phrases like "view filter", "color code elements", "graphic overrides", "color by parameter",
  "filter by phase", "highlight elements", "visibility graphics", "create filters", "apply
  filters to views", or any request to visually distinguish elements in Revit views.
---

# Create View Filters & Graphic Overrides

Create and apply Revit view filters for:

**"$ARGUMENTS"**

## Before writing code

Clarify (one question if ambiguous):
1. **Which parameter** drives the filter? (e.g. Phase, Status, Department, Type Mark, Workset)
2. **Which views** should filters be applied to? (Active view, all floor plans, specific view names?)
3. **What colors/overrides?** (Specific colors per value, or just hide/show?)

---

## Script: Create parametric filter and apply to views

This script creates ParameterFilterElement rules and applies graphic overrides. **Modifies the model.**

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

import System
from System.Collections.Generic import List

doc  = DocumentManager.Instance.CurrentDBDocument

# ── Configuration ────────────────────────────────────────────────────────────
# Categories the filter applies to
TARGET_CATEGORIES = [
    BuiltInCategory.OST_Walls,
    BuiltInCategory.OST_Floors,
    BuiltInCategory.OST_Roofs,
]

# Parameter to filter on (exact name, case-sensitive)
FILTER_PARAM_NAME = "Phase Created"   # or use a BuiltInParameter below

# Filter rules: each entry = (filter_name, param_value_to_match, RGB_color_tuple)
# The script creates one filter per rule.
FILTER_RULES = [
    ("Phase - New Construction", "New Construction", (0, 150, 255)),   # blue
    ("Phase - Existing",         "Existing",         (180, 180, 180)), # grey
    ("Phase - Demolished",       "Demolished",       (255, 80, 80)),   # red
]

# Apply filters to ALL floor plan views? Set to False to use APPLY_TO_VIEW_NAMES list.
APPLY_TO_ALL_FLOOR_PLANS = True
APPLY_TO_VIEW_NAMES = ["Level 1", "Level 2"]   # used only if above is False
# ─────────────────────────────────────────────────────────────────────────────

def make_color(r, g, b):
    return Color(r, g, b)

def get_param_id(doc, cat_ids, param_name):
    """Return the ElementId of a shared/project parameter by name."""
    # Try BuiltInParameter first for common phase params
    bp_map = {
        "Phase Created": BuiltInParameter.PHASE_CREATED,
        "Phase Demolished": BuiltInParameter.PHASE_DEMOLISHED,
    }
    if param_name in bp_map:
        return ElementId(bp_map[param_name])
    # Fall back to iterating shared parameters
    binding_map = doc.ParameterBindings
    it = binding_map.ForwardIterator()
    while it.MoveNext():
        defn = it.Key
        if defn.Name == param_name:
            # Get ElementId via ParameterElement
            for pe in FilteredElementCollector(doc).OfClass(ParameterElement).ToElements():
                if pe.GetDefinition().Name == param_name:
                    return pe.Id
    return None

try:
    # Collect category IDs
    cat_id_list = List[ElementId]()
    for bic in TARGET_CATEGORIES:
        cat = doc.Settings.Categories.get_Item(bic)
        if cat:
            cat_id_list.Add(cat.Id)

    # Get the target views
    all_views = list(FilteredElementCollector(doc).OfClass(View).ToElements())
    if APPLY_TO_ALL_FLOOR_PLANS:
        target_views = [v for v in all_views
                        if v.ViewType == ViewType.FloorPlan and not v.IsTemplate]
    else:
        target_views = [v for v in all_views
                        if v.Name in APPLY_TO_VIEW_NAMES and not v.IsTemplate]

    param_id = get_param_id(doc, cat_id_list, FILTER_PARAM_NAME)
    if not param_id:
        OUT = "ERROR: Could not find parameter '" + FILTER_PARAM_NAME + "'. Check the exact name."
    else:
        created_filters = []
        TransactionManager.Instance.EnsureInTransaction(doc)

        for filter_name, match_value, (r, g, b) in FILTER_RULES:
            # Check if filter already exists — delete and recreate to update
            existing = [f for f in FilteredElementCollector(doc)
                            .OfClass(ParameterFilterElement).ToElements()
                        if f.Name == filter_name]
            for old in existing:
                doc.Delete(old.Id)

            # Build the filter rule: parameter equals string value
            rule = ParameterFilterRuleFactory.CreateEqualsRule(
                param_id,
                match_value,
                True   # case-sensitive
            )
            elem_filter = ElementParameterFilter(rule)

            # Create the ParameterFilterElement
            pf = ParameterFilterElement.Create(doc, filter_name, cat_id_list, elem_filter)

            # Build graphic overrides
            ogs = OverrideGraphicSettings()
            col = make_color(r, g, b)
            ogs.SetProjectionLineColor(col)
            ogs.SetSurfaceForegroundPatternColor(col)
            ogs.SetSurfaceForegroundPatternVisible(True)

            # Apply to each target view
            for view in target_views:
                try:
                    view.AddFilter(pf.Id)
                    view.SetFilterOverrides(pf.Id, ogs)
                    view.SetFilterVisibility(pf.Id, True)
                except Exception:
                    pass  # skip views that don't support the filter

            created_filters.append(filter_name)

        TransactionManager.Instance.TransactionTaskDone()
        OUT = {
            "Filters created": created_filters,
            "Applied to views": [v.Name for v in target_views],
            "Parameter used": FILTER_PARAM_NAME
        }

except Exception as e:
    TransactionManager.Instance.ForceCloseTransaction()
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Script: List existing filters in the model (read-only)

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

try:
    filters = list(FilteredElementCollector(doc)
                   .OfClass(ParameterFilterElement).ToElements())
    result = []
    for f in filters:
        cats = [doc.Settings.Categories.get_Item(cid).Name
                for cid in f.GetCategories()
                if doc.Settings.Categories.get_Item(cid) is not None]
        result.append({"Name": f.Name, "Categories": ", ".join(cats)})
    OUT = result if result else "No view filters found in this model."
except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Common filter parameters

| Goal                        | Parameter name          | BuiltInParameter               |
|-----------------------------|-------------------------|--------------------------------|
| Filter by phase             | `Phase Created`         | `PHASE_CREATED`                |
| Filter by phase demolished  | `Phase Demolished`      | `PHASE_DEMOLISHED`             |
| Filter by workset           | `Workset`               | `ELEM_PARTITION_PARAM`         |
| Filter by type mark         | `Type Mark`             | `WINDOW_TYPE_ID` (doors/wins)  |
| Filter by department        | `Department`            | `ROOM_DEPARTMENT`              |
| Filter by design option     | `Design Option`         | `DESIGN_OPTION_ID`             |

## Customization tips

- **Multiple categories**: add more `BuiltInCategory` entries to `TARGET_CATEGORIES`
- **Solid fill**: also call `ogs.SetSurfaceForegroundPatternId(solid_fill_pattern_id)` with the filled-region pattern Id
- **Halftone / transparency**: `ogs.SetHalftone(True)` or `ogs.SetSurfaceTransparency(50)`
- **String vs Integer parameters**: use `CreateEqualsRule` for strings, `CreateGreaterOrEqualRule` for numbers
