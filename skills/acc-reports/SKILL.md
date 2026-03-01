---
name: acc-reports
description: >
  Export ACC project data to CSV or Excel — issues, RFIs, document logs, clash summaries,
  transmittals, and project activity reports — using the APS REST API with Python. Use
  this skill when a user needs to report on ACC project health, export an issues register,
  pull RFI status into a spreadsheet, generate a document transmittal log, create a clash
  trend report, or produce any data export from Autodesk Construction Cloud. Trigger on
  phrases like "export issues", "issues report", "RFI report", "ACC report", "export from
  ACC", "document log", "clash report", "ACC data export", "project status report", "pull
  data from ACC", "ACC Excel export", "transmittal log", or any request to extract or
  report on data stored in Autodesk Construction Cloud.
---

# ACC Data Reports & Exports

Export ACC project data for:

**"$ARGUMENTS"**

## Prerequisites

- APS credentials configured (see `acc-api-setup` skill)
- For Issues/RFIs: **3-legged token** required
- For Docs/Clash data: 2-legged token is sufficient
- `pip install requests openpyxl`

---

## Script 1: Export all ACC Issues to CSV

Issues API requires a 3-legged token (user context).

```python
import os, csv, requests
from datetime import datetime

# ── Configuration ─────────────────────────────────────────────────────────────
PROJECT_ID  = os.environ.get("APS_PROJECT_ID_RAW")   # WITHOUT b. prefix
OUTPUT_PATH = r"C:\Reports\acc_issues_export.csv"

BASE_ISSUES = "https://developer.api.autodesk.com/construction/issues/v1"
# ─────────────────────────────────────────────────────────────────────────────

def get_all_issues(client, project_id):
    """Fetch all issues from ACC using pagination."""
    all_issues = []
    offset     = 0
    limit      = 100

    while True:
        url    = f"{BASE_ISSUES}/projects/{project_id}/issues"
        params = {"limit": limit, "offset": offset,
                  "filterType": "all"}   # include all statuses
        data   = client.get(url, params=params)

        issues = data.get("results", [])
        total  = data.get("pagination", {}).get("totalResults", 0)

        all_issues.extend(issues)
        offset += limit

        if offset >= total or not issues:
            break

    return all_issues, total

def issues_to_csv(issues, output_path):
    """Write issues list to CSV."""
    if not issues:
        print("No issues found.")
        return

    os.makedirs(os.path.dirname(output_path), exist_ok=True)

    fieldnames = [
        "Issue Number", "Title", "Status", "Assigned To",
        "Due Date", "Created By", "Created Date",
        "Root Cause", "Type", "Sub-type",
        "Location", "Description", "Closed Date",
    ]

    with open(output_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames, extrasaction="ignore")
        writer.writeheader()

        for issue in issues:
            attrs = issue.get("attributes", {})
            writer.writerow({
                "Issue Number":  attrs.get("displayId", ""),
                "Title":         attrs.get("title", ""),
                "Status":        attrs.get("status", ""),
                "Assigned To":   (attrs.get("assignedTo", {}) or {}).get("displayName", ""),
                "Due Date":      attrs.get("dueDate", ""),
                "Created By":    (attrs.get("createdBy", {}) or {}).get("displayName", ""),
                "Created Date":  attrs.get("createdAt", ""),
                "Root Cause":    attrs.get("rootCauseLabel", ""),
                "Type":          attrs.get("issueTypeLabel", ""),
                "Sub-type":      attrs.get("issueSubTypeLabel", ""),
                "Location":      attrs.get("locationDetails", ""),
                "Description":   attrs.get("description", "")[:200] if attrs.get("description") else "",
                "Closed Date":   attrs.get("closedAt", ""),
            })

    print(f"Issues exported: {len(issues)} → {output_path}")

if __name__ == "__main__":
    # client must use a 3-legged token
    issues, total = get_all_issues(client, PROJECT_ID)
    print(f"Fetched {len(issues)} of {total} issues")
    issues_to_csv(issues, OUTPUT_PATH)
```

---

## Script 2: Issues summary dashboard (pivot by status, type, assignee)

```python
import os, csv
from collections import Counter, defaultdict

def issues_summary(issues, output_path=r"C:\Reports\acc_issues_summary.csv"):
    """Generate a pivot summary of issues by status, type, and assignee."""

    status_count  = Counter()
    type_count    = Counter()
    assignee_count = Counter()
    overdue_count = 0

    today = datetime.now().date()

    for issue in issues:
        attrs = issue.get("attributes", {})

        status   = attrs.get("status", "unknown")
        iss_type = attrs.get("issueTypeLabel", "unknown")
        assignee = (attrs.get("assignedTo", {}) or {}).get("displayName", "Unassigned")
        due_str  = attrs.get("dueDate", "")

        status_count[status]     += 1
        type_count[iss_type]     += 1
        assignee_count[assignee] += 1

        if due_str and status not in ("closed", "void"):
            try:
                due_date = datetime.fromisoformat(due_str.split("T")[0]).date()
                if due_date < today:
                    overdue_count += 1
            except Exception:
                pass

    os.makedirs(os.path.dirname(output_path), exist_ok=True)

    with open(output_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)

        writer.writerow(["=== ISSUES BY STATUS ==="])
        writer.writerow(["Status", "Count"])
        for status, count in status_count.most_common():
            writer.writerow([status, count])

        writer.writerow([])
        writer.writerow(["=== ISSUES BY TYPE ==="])
        writer.writerow(["Type", "Count"])
        for typ, count in type_count.most_common():
            writer.writerow([typ, count])

        writer.writerow([])
        writer.writerow(["=== ISSUES BY ASSIGNEE ==="])
        writer.writerow(["Assignee", "Count"])
        for assignee, count in assignee_count.most_common():
            writer.writerow([assignee, count])

        writer.writerow([])
        writer.writerow(["=== SUMMARY ==="])
        writer.writerow(["Total Issues",   sum(status_count.values())])
        writer.writerow(["Open",           status_count.get("open", 0)])
        writer.writerow(["In Review",      status_count.get("in_review", 0)])
        writer.writerow(["Closed",         status_count.get("closed", 0)])
        writer.writerow(["Overdue (open)", overdue_count])

    print(f"Summary saved: {output_path}")

# Usage:
# issues_summary(issues)
```

---

## Script 3: Export ACC Docs file register to CSV

```python
import os, csv

BASE_URL = "https://developer.api.autodesk.com"

def export_doc_register(client, hub_id, project_id, output_path=r"C:\Reports\acc_doc_register.csv"):
    """Export a flat file register of all documents in ACC Docs."""

    # Get top folders
    folders = client.get(
        f"{BASE_URL}/project/v1/hubs/{hub_id}/projects/{project_id}/topFolders"
    ).get("data", [])

    all_files = []

    def traverse(folder_id, folder_path):
        url  = f"{BASE_URL}/data/v1/projects/{project_id}/folders/{folder_id}/contents"
        page = url
        while page:
            data  = client.get(page)
            items = data.get("data", [])
            next_ = data.get("links", {}).get("next", {})
            for item in items:
                attrs = item["attributes"]
                if item["type"] == "folders":
                    traverse(item["id"], folder_path + "/" + attrs["name"])
                elif item["type"] == "items":
                    all_files.append({
                        "File Name":     attrs.get("displayName", ""),
                        "Folder Path":   folder_path,
                        "Item ID":       item["id"],
                        "Last Modified": attrs.get("lastModifiedTime", ""),
                        "Modified By":   attrs.get("lastModifiedUserName", ""),
                        "Created":       attrs.get("createTime", ""),
                        "Created By":    attrs.get("createUserName", ""),
                    })
            page = next_.get("href") if next_ else None

    for folder in folders:
        traverse(folder["id"], folder["attributes"]["name"])

    os.makedirs(os.path.dirname(output_path), exist_ok=True)
    with open(output_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=list(all_files[0].keys()) if all_files else [])
        writer.writeheader()
        writer.writerows(all_files)

    print(f"Doc register: {len(all_files)} files → {output_path}")
    return all_files
```

---

## Script 4: Clash trend report (compare current vs previous version)

```python
import csv, os
from datetime import datetime

BASE_MC = "https://developer.api.autodesk.com/modelcoordination/modelset/v3"

def clash_trend_report(client, container_id, model_set_id,
                        output_path=r"C:\Reports\acc_clash_trend.csv"):
    """Compare clash counts across all model set versions to show trend."""

    url   = f"{BASE_MC}/containers/{container_id}/modelsets/{model_set_id}/versions"
    data  = client.get(url)
    versions = sorted(
        data.get("results", []),
        key=lambda v: v.get("versionNumber", 0)
    )

    os.makedirs(os.path.dirname(output_path), exist_ok=True)

    with open(output_path, "w", newline="") as f:
        writer = csv.writer(f)
        writer.writerow(["Version", "Date", "Total Clashes", "Delta vs Previous"])

        prev_count = None
        for v in versions:
            count = v.get("clashCount", 0)
            delta = ""
            if prev_count is not None:
                diff  = count - prev_count
                delta = ("+" if diff > 0 else "") + str(diff)
            writer.writerow([
                "v" + str(v.get("versionNumber", "")),
                v.get("createTime", "")[:10],
                count,
                delta,
            ])
            prev_count = count

    print(f"Clash trend saved: {output_path}")

# Usage:
# clash_trend_report(client, CONTAINER_ID, MODEL_SET_ID)
```

---

## Combined project health report

```python
def full_project_report(client, hub_id, project_id, container_id, model_set_id):
    """Generate a complete ACC project health snapshot."""
    from datetime import datetime

    report = {
        "Generated":    datetime.now().strftime("%Y-%m-%d %H:%M"),
        "Project ID":   project_id,
    }

    # Issues
    try:
        issues, total = get_all_issues(client, project_id.replace("b.", ""))
        open_issues   = sum(1 for i in issues
                            if i.get("attributes", {}).get("status") == "open")
        report["Total Issues"]  = total
        report["Open Issues"]   = open_issues
        report["Closed Issues"] = total - open_issues
    except Exception as e:
        report["Issues Error"] = str(e)

    # Docs
    try:
        files = export_doc_register(client, hub_id, project_id,
                                    output_path=r"C:\Reports\doc_register.csv")
        report["Total Documents"] = len(files)
    except Exception as e:
        report["Docs Error"] = str(e)

    # Clashes
    try:
        versions = get_model_set_versions(client, container_id, model_set_id)
        if versions:
            report["Latest Clash Count"] = versions[0]["clash_count"]
            report["Latest Model Set Version"] = versions[0]["version_num"]
    except Exception as e:
        report["Clash Error"] = str(e)

    return report
```

---

## Quick reference: ACC report types and which token they need

| Report                 | API                        | Token     | Scope          |
|------------------------|----------------------------|-----------|----------------|
| Issues register        | Issues API v1              | 3-legged  | data:read      |
| Document log           | Data Management API v1     | 2-legged  | data:read      |
| Clash summary          | Model Coordination API v3  | 2-legged  | data:read      |
| Project list           | Project API v1             | 2-legged  | account:read   |
| Transmittal log        | Construction Transmittals  | 2-legged  | data:read      |
| Sheets register        | Sheets API v1              | 2-legged  | data:read      |
