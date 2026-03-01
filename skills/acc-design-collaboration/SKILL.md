---
name: acc-design-collaboration
description: >
  Guide Revit ↔ ACC Design Collaboration workflows — setting up cloud worksharing, creating
  and consuming design packages, shared views, Design Collaboration setup, model publishing,
  and Revit cloud model best practices. Also covers the API for querying shared views and
  design package status. Use this skill when a user needs help with Revit cloud models,
  BIM 360 Design / ACC Design Collaboration, creating or receiving design packages,
  activating models for coordination, or troubleshooting cloud worksharing issues. Trigger
  on phrases like "Design Collaboration", "cloud worksharing", "Revit cloud model", "design
  package", "shared views", "consume package", "publish model to ACC", "BIM 360 Design",
  "Revit cloud", "cloud model", "collaboration workflow", "ACC Revit workflow", or any
  request involving the Revit-to-ACC publishing and collaboration pipeline.
---

# ACC Design Collaboration — Revit Cloud Workflow

Set up and manage Design Collaboration for:

**"$ARGUMENTS"**

## Overview

ACC Design Collaboration connects Revit teams working on cloud-hosted models. The workflow has two modes:

| Mode                        | When to use                                                  |
|-----------------------------|--------------------------------------------------------------|
| **Cloud Worksharing**       | Multiple Revit users on the same model simultaneously        |
| **Design Packages**         | Sharing a frozen snapshot of your model with other disciplines|

---

## Part 1: Cloud Worksharing Setup (Revit UI — no API needed)

### Step-by-step: Moving a local model to ACC

1. **Open the Revit model** you want to migrate.
2. Go to **Collaborate** tab → **Collaborate** button.
3. Select **"In BIM 360 / Autodesk Construction Cloud"**.
4. Sign in with your Autodesk ID if prompted.
5. Choose the **ACC project** and **folder** — place it in the Design Collaboration folder (usually `Project Files > [Discipline]`).
6. Click **OK** — Revit saves the model to ACC and creates a local cache file.
7. All team members open the model from ACC using **Open → BIM 360 / ACC** in Revit.

### Workset best practices for cloud models
- Keep **"Shared Levels and Grids"** on its own workset — other disciplines will borrow from it.
- Each discipline should own one workset (Architecture, Structure, MEP, etc.)
- All team members should **Synchronize with Central (SWC)** at start and end of each session.
- Use **Relinquish All Mine** before closing.
- Never use **Save As** on a cloud model — it detaches from the cloud.

### Who should use Cloud Worksharing?
- Same discipline, multiple users (e.g. 3 architects in the same Revit model)
- Teams working concurrently with live updates

---

## Part 2: Design Packages — Sharing with other disciplines

Design Packages let you send a **frozen snapshot** of your model to other teams for reference.

### Create a Design Package (Revit UI)

1. In Revit, go to **Collaborate** tab → **Design Collaboration**.
2. In the Design Collaboration panel, select your project.
3. Under **Packages**, click **Create Package**.
4. Name the package (e.g. "ARCH-P01-2024-03-01" — use your project's naming convention).
5. Select the **views/models** to include.
6. Add a description / change summary.
7. Click **Publish**.
8. ACC sends a notification to the receiving teams.

### Consume a package from another discipline (Revit UI)

1. In Revit → **Collaborate** → **Design Collaboration**.
2. Under **Incoming Packages**, you will see new packages from other teams.
3. Click **Review** to see what changed.
4. Click **Accept** to update your local reference files.
5. The incoming model appears as a linked Revit file in your project.

### Design Package naming convention (recommended)

```
{Discipline}-{Status}-{Date}-{Revision}
Examples:
  ARCH-WIP-2024-03-01-R01         ← work in progress share
  STRUCT-S1-2024-03-15-C01        ← suitable for coordination
  MEP-S2-2024-04-01-C03           ← suitable for information
```

---

## Part 3: Shared Views

Shared Views let you publish a **non-downloadable 3D/2D view** to ACC for review by non-Revit users (clients, contractors, planners).

### Create a Shared View (Revit UI)

1. Open the 3D view or sheet you want to share.
2. Go to **Collaborate** → **Share**.
3. Choose **Send Link** — Revit publishes the view to ACC Docs.
4. Copy the link and send to reviewers. They open it in a browser with no Revit needed.
5. Shared Views expire after **30 days** by default (configurable).

---

## Part 4: API — Query Shared Views and Design Package status

### Get shared views via APS API

```python
import os, requests

# Shared Views are stored in ACC Docs — query via Data Management API
# They appear in the "Shared Views" folder in the project

BASE_URL   = "https://developer.api.autodesk.com"
PROJECT_ID = os.environ.get("APS_PROJECT_ID")  # with b. prefix

def find_shared_views_folder(client, hub_id, project_id):
    """Find the Shared Views folder in an ACC project."""
    url   = f"{BASE_URL}/project/v1/hubs/{hub_id}/projects/{project_id}/topFolders"
    data  = client.get(url)
    for folder in data.get("data", []):
        if "shared" in folder["attributes"]["name"].lower():
            return folder["id"]
    return None

def list_shared_views(client, project_id, folder_id):
    """List all shared views in the Shared Views folder."""
    url  = f"{BASE_URL}/data/v1/projects/{project_id}/folders/{folder_id}/contents"
    data = client.get(url)
    views = []
    for item in data.get("data", []):
        if item["type"] == "items":
            attrs = item["attributes"]
            views.append({
                "Name":          attrs.get("displayName", ""),
                "Item ID":       item["id"],
                "Created":       attrs.get("createTime", ""),
                "Last Modified": attrs.get("lastModifiedTime", ""),
                "Created By":    attrs.get("createUserName", ""),
            })
    return views
```

### Check Design Collaboration package status via Revit API

ACC Design Collaboration doesn't have a fully public REST API for packages (as of 2025), but you can check package status using the **Revit API** inside Dynamo:

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc = DocumentManager.Instance.CurrentDBDocument

try:
    # Check if this model is a cloud model
    cloud_path = doc.GetCloudModelPath()
    if cloud_path:
        OUT = {
            "Is Cloud Model":  True,
            "Project GUID":    str(cloud_path.GetProjectGUID()),
            "Model GUID":      str(cloud_path.GetModelGUID()),
            "Region":          cloud_path.Region if hasattr(cloud_path, "Region") else "N/A",
        }
    else:
        OUT = {"Is Cloud Model": False}

except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Part 5: Revit publish to ACC from Dynamo (automatic sync)

```python
import clr
clr.AddReference('RevitServices')
from RevitServices.Persistence import DocumentManager
from RevitServices.Transactions import TransactionManager
clr.AddReference('RevitAPI')
from Autodesk.Revit.DB import *

doc   = DocumentManager.Instance.CurrentDBDocument
uidoc = DocumentManager.Instance.CurrentUIApplication.ActiveUIDocument

try:
    if not doc.IsWorkshared:
        OUT = "Model is not workshared — cannot synchronise."
    else:
        # Build sync options
        opts = SynchronizeWithCentralOptions()
        opts.SetRelinquishOptions(RelinquishOptions(True))  # relinquish all on sync
        opts.SaveLocalBefore = True
        opts.SaveLocalAfter  = True
        opts.Comment         = "Auto-sync via Dynamo script"

        doc.SynchronizeWithCentral(TransactWithCentralOptions(), opts)
        OUT = "Synchronised with Central successfully."

except Exception as e:
    import traceback
    OUT = "ERROR: " + str(e) + "\n" + traceback.format_exc()
```

---

## Common Design Collaboration problems & fixes

| Problem                               | Fix                                                              |
|---------------------------------------|------------------------------------------------------------------|
| Can't open model — "not found"        | Check you're signed into the correct Autodesk account            |
| "Central model out of date"           | Another user is saving — wait and retry SWC                      |
| Missing linked files after sync       | Reload Links from cloud path, not local path                     |
| Package not visible to other team     | Check permissions — receiving team needs at least "View" access  |
| Model performance slow on cloud       | Use "Specify" workset opening — don't open all worksets          |
| Changes not reflecting in packages    | Create a new package — packages are snapshots, not live links    |
| SWC failing with ownership errors     | Run "Relinquish All Mine" then retry SWC                         |

---

## Design Collaboration checklist for project start

- [ ] Create ACC project with correct permissions (Design Collaboration role assigned)
- [ ] Set up discipline folders in ACC Docs (Architecture, Structure, MEP, etc.)
- [ ] Migrate each model to ACC cloud via Collaborate → Collaborate
- [ ] Establish workset naming convention and assign ownership
- [ ] Configure Design Collaboration settings: who can create packages, package naming
- [ ] Define package sharing frequency (weekly sprints, milestone-based, etc.)
- [ ] Test that all disciplines can consume each other's packages
- [ ] Set up Model Coordination model set and activate discipline models
