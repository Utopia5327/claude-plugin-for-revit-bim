---
name: acc-coordinate
description: >
  Work with Autodesk Construction Cloud (ACC) Coordinate — list and inspect model sets,
  retrieve clash results, create clash issues, manage model activation for coordination,
  and export coordination reports via the APS Model Coordination API. Use this skill when
  a user needs to automate clash detection workflows, pull clash data from ACC for reporting,
  create ACC issues from clashes, check which models are active in a model set, or integrate
  ACC coordination data with other tools. Trigger on phrases like "ACC Coordinate", "model
  coordination", "model set", "clash results", "clash data", "ACC clashes", "coordination
  space", "activated models", "clash issues", "federated model", "coordination report",
  "BIM coordination", or any request to work with ACC's model coordination module via API.
  Also offer this when users want to export or report on clash data from ACC.
---

# ACC Coordinate — Model Coordination API

Automate ACC Coordinate workflows for:

**"$ARGUMENTS"**

## Prerequisites

- APS credentials configured (see `acc-api-setup` skill)
- Project container ID = ACC project ID **without** the `b.` prefix
- `pip install requests`

---

## API base URL and key concepts

```
Base:  https://developer.api.autodesk.com/modelcoordination/modelset/v3
Auth:  2-legged token (data:read scope)
```

**Key objects:**
- **Container**: the coordination space for a project (ID = project ID without `b.`)
- **Model Set**: a named collection of activated Revit/IFC models grouped for clash testing
- **Clash Set**: the result of a clash test between two model sets or within one
- **Clash Group**: a group of related clashes (e.g. all ducts vs all beams)
- **Clash Instance**: a single clash between two specific elements

---

## Script 1: List all model sets in a project

```python
import os, requests

CONTAINER_ID = os.environ.get("APS_CONTAINER_ID")  # project ID without b.
BASE_MC = "https://developer.api.autodesk.com/modelcoordination/modelset/v3"

def list_model_sets(client, container_id):
    """List all model sets (coordination spaces) in an ACC project."""
    url  = f"{BASE_MC}/containers/{container_id}/modelsets"
    data = client.get(url)
    model_sets = []
    for ms in data.get("results", []):
        model_sets.append({
            "Model Set Name": ms.get("name", ""),
            "Model Set ID":   ms.get("modelSetId", ""),
            "Created":        ms.get("createTime", ""),
            "Status":         ms.get("status", ""),
            "Model Count":    len(ms.get("modelSetVersions", [{}])[-1:][0].get("models", []))
                              if ms.get("modelSetVersions") else 0,
        })
    return model_sets

if __name__ == "__main__":
    # client = APSClient(...)
    model_sets = list_model_sets(client, CONTAINER_ID)
    for ms in model_sets:
        print(f"  {ms['Model Set Name']:30} ID: {ms['Model Set ID']}")
```

---

## Script 2: Get clash results for a model set

```python
BASE_MC = "https://developer.api.autodesk.com/modelcoordination/modelset/v3"

def get_model_set_versions(client, container_id, model_set_id):
    """Get all versions of a model set (each version = a clash test run)."""
    url  = f"{BASE_MC}/containers/{container_id}/modelsets/{model_set_id}/versions"
    data = client.get(url)
    versions = []
    for v in data.get("results", []):
        versions.append({
            "version_id":    v.get("modelSetVersionId", ""),
            "version_num":   v.get("versionNumber", 0),
            "created":       v.get("createTime", ""),
            "clash_count":   v.get("clashCount", 0),
            "status":        v.get("status", ""),
        })
    # Sort by version number descending (latest first)
    return sorted(versions, key=lambda x: -x["version_num"])

def get_clash_groups(client, container_id, model_set_id, version_id):
    """Get grouped clashes for a model set version."""
    url  = f"{BASE_MC}/containers/{container_id}/modelsets/{model_set_id}/versions/{version_id}/clashes/grouped"
    data = client.get(url)
    groups = []
    for group in data.get("groups", []):
        groups.append({
            "Clash Group":    group.get("clashGroupId", ""),
            "Discipline A":   group.get("documentGroupA", {}).get("name", ""),
            "Discipline B":   group.get("documentGroupB", {}).get("name", ""),
            "Count":          group.get("clashCount", 0),
            "Status":         group.get("status", "open"),
        })
    return sorted(groups, key=lambda x: -x["Count"])

def get_clash_instances(client, container_id, model_set_id, version_id, group_id, limit=100):
    """Get individual clash instances in a clash group."""
    url    = (f"{BASE_MC}/containers/{container_id}/modelsets/{model_set_id}"
              f"/versions/{version_id}/clashes")
    params = {"groupId": group_id, "limit": limit}
    data   = client.get(url, params=params)
    clashes = []
    for clash in data.get("clashes", []):
        clashes.append({
            "Clash ID":    clash.get("clashId", ""),
            "Status":      clash.get("status", ""),
            "Element A ID": clash.get("ldId", ""),
            "Element B ID": clash.get("rdId", ""),
            "Position X":  clash.get("clashPoint", {}).get("x", 0),
            "Position Y":  clash.get("clashPoint", {}).get("y", 0),
            "Position Z":  clash.get("clashPoint", {}).get("z", 0),
        })
    return clashes

if __name__ == "__main__":
    # Get latest version
    versions = get_model_set_versions(client, CONTAINER_ID, MODEL_SET_ID)
    latest   = versions[0]
    print(f"Latest version: v{latest['version_num']} — {latest['clash_count']} clashes")

    # Get clash groups (discipline pairs)
    groups = get_clash_groups(client, CONTAINER_ID, MODEL_SET_ID, latest["version_id"])
    for g in groups[:10]:
        print(f"  {g['Discipline A']:25} vs {g['Discipline B']:25} → {g['Count']} clashes")
```

---

## Script 3: Export clash data to CSV

```python
import csv, os
from datetime import datetime

def export_clashes_to_csv(client, container_id, model_set_id, output_path=None):
    """Export all clash groups and totals to a CSV report."""

    if not output_path:
        ts = datetime.now().strftime("%Y%m%d_%H%M")
        output_path = f"C:\\Reports\\acc_clashes_{ts}.csv"

    # Get latest version
    versions = get_model_set_versions(client, container_id, model_set_id)
    if not versions:
        print("No versions found.")
        return

    latest_version = versions[0]
    groups = get_clash_groups(client, container_id, model_set_id, latest_version["version_id"])

    os.makedirs(os.path.dirname(output_path), exist_ok=True)

    with open(output_path, "w", newline="") as f:
        writer = csv.writer(f)
        writer.writerow([
            "Report Date", datetime.now().strftime("%Y-%m-%d %H:%M"),
            "Model Set ID", model_set_id,
            "Version", latest_version["version_num"],
        ])
        writer.writerow([])
        writer.writerow(["Clash Group", "Discipline A", "Discipline B", "Count", "Status"])

        total = 0
        for g in groups:
            writer.writerow([
                g["Clash Group"],
                g["Discipline A"],
                g["Discipline B"],
                g["Count"],
                g["Status"],
            ])
            total += g["Count"]

        writer.writerow([])
        writer.writerow(["TOTAL", "", "", total, ""])

    print(f"Clash report saved to: {output_path} ({total} total clashes)")
    return output_path
```

---

## Script 4: List activated models in a model set

```python
def get_activated_models(client, container_id, model_set_id):
    """List all Revit/IFC models activated in the latest model set version."""
    versions = get_model_set_versions(client, container_id, model_set_id)
    if not versions:
        return []

    latest_v_id = versions[0]["version_id"]
    url  = f"{BASE_MC}/containers/{container_id}/modelsets/{model_set_id}/versions/{latest_v_id}"
    data = client.get(url)

    models = []
    for model in data.get("models", []):
        models.append({
            "Model Name":   model.get("name", ""),
            "Document ID":  model.get("documentId", ""),
            "Version URN":  model.get("versionUrn", ""),
            "Discipline":   model.get("documentGroup", {}).get("name", ""),
            "Status":       model.get("status", ""),
        })
    return models
```

---

## What to tell the user about ACC Coordinate

### Model set setup (no API needed — ACC UI)
1. Go to ACC → **Coordinate** → **Model Coordination**
2. Create a new Model Set, give it a name (e.g. "Full Building — All Disciplines")
3. Select folders from ACC Docs to include (Revit and IFC models from each discipline)
4. **Activate** the models in the set
5. Run a clash test — ACC generates results within minutes to hours

### Model activation best practices
- Activate only **coordination models** (not design working files)
- Use **IFC** exports for cross-authoring-tool coordination
- Separate model sets by zone or level for complex projects
- Re-run clash tests after every model update/publish

### Clash triage workflow
1. Export clash groups CSV (Script 3 above)
2. Sort by count → address highest-density clashes first
3. Assign clashes to discipline owners as ACC Issues
4. Track resolution in weekly coordination meetings
5. Re-run clash test and compare delta to previous version
