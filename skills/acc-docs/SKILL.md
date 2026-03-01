---
name: acc-docs
description: >
  Manage Autodesk Construction Cloud (ACC) Docs — list folders and files, upload documents,
  download files, create folder structures, manage transmittals, reviews, and packages
  via the APS Data Management API and ACC Docs REST API. Use this skill when a user needs
  to automate document control workflows in ACC: bulk uploading drawing sets, creating
  folder structures, listing published files, downloading specific versions, setting up
  transmittals, creating document packages, or writing reports of what's in their ACC Docs
  library. Trigger on phrases like "ACC Docs", "upload to ACC", "download from ACC",
  "list files in ACC", "document control", "ACC folder", "transmittal", "document package",
  "file versions", "publish drawings", "drawing register", "ACC file management", or any
  request to read, write, or organise files in Autodesk Construction Cloud Docs.
---

# ACC Docs — Document Management via APS API

Manage ACC Docs for:

**"$ARGUMENTS"**

## Prerequisites

- APS credentials set up (use `acc-api-setup` skill first)
- `APSClient` class available (from acc-api-setup)
- Hub ID and Project ID known (run the discovery script in acc-api-setup)
- `pip install requests`

---

## Script 1: List all files in a folder (recursive)

```python
import os, requests

# ── Configuration ─────────────────────────────────────────────────────────────
HUB_ID     = os.environ.get("APS_HUB_ID",     "b.YOUR_HUB_ID")
PROJECT_ID = os.environ.get("APS_PROJECT_ID", "b.YOUR_PROJECT_ID")   # with b. prefix

BASE_URL = "https://developer.api.autodesk.com"
# ─────────────────────────────────────────────────────────────────────────────

def get_top_folders(client, hub_id, project_id):
    """Get the top-level folders in an ACC project."""
    url  = f"{BASE_URL}/project/v1/hubs/{hub_id}/projects/{project_id}/topFolders"
    data = client.get(url)
    return data.get("data", [])

def list_folder_contents(client, project_id, folder_id, depth=0, results=None):
    """Recursively list all items in a folder."""
    if results is None:
        results = []
    url  = f"{BASE_URL}/data/v1/projects/{project_id}/folders/{folder_id}/contents"

    page_url = url
    while page_url:
        data    = client.get(page_url)
        items   = data.get("data", [])
        links   = data.get("links", {})

        for item in items:
            item_type = item["type"]   # "folders" or "items"
            attrs     = item["attributes"]
            indent    = "  " * depth

            if item_type == "folders":
                results.append({
                    "type":     "Folder",
                    "name":     attrs["name"],
                    "id":       item["id"],
                    "depth":    depth,
                    "path":     indent + "📁 " + attrs["name"],
                })
                # Recurse into subfolder
                list_folder_contents(client, project_id, item["id"], depth + 1, results)

            elif item_type == "items":
                # Get the latest version info
                version = item.get("relationships", {}).get("tip", {}).get("data", {})
                results.append({
                    "type":        "File",
                    "name":        attrs["displayName"],
                    "id":          item["id"],
                    "version_id":  version.get("id", ""),
                    "depth":       depth,
                    "path":        indent + "📄 " + attrs["displayName"],
                })

        # Handle pagination
        next_link = links.get("next", {})
        page_url  = next_link.get("href") if next_link else None

    return results

# ── Run ───────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    # client = APSClient(...)  # from acc-api-setup

    folders = get_top_folders(client, HUB_ID, PROJECT_ID)
    all_files = []

    for folder in folders:
        print(f"\n=== {folder['attributes']['name']} ===")
        contents = list_folder_contents(client, PROJECT_ID, folder["id"])
        for item in contents:
            print(item["path"])
        all_files.extend(contents)

    print(f"\nTotal items: {len(all_files)}")
```

---

## Script 2: List file versions and download a specific version

```python
import os, requests

BASE_URL = "https://developer.api.autodesk.com"

def get_versions(client, project_id, item_id):
    """List all versions of a file."""
    url  = f"{BASE_URL}/data/v1/projects/{project_id}/items/{item_id}/versions"
    data = client.get(url)
    versions = []
    for v in data.get("data", []):
        attrs = v["attributes"]
        versions.append({
            "version_number": attrs.get("versionNumber", ""),
            "version_id":     v["id"],
            "file_name":      attrs.get("name", ""),
            "file_size_mb":   round(attrs.get("storageSize", 0) / 1048576, 2),
            "last_modified":  attrs.get("lastModifiedTime", ""),
            "created_by":     attrs.get("lastModifiedUserName", ""),
        })
    return versions

def download_version(client, project_id, version_id, output_path):
    """Download a specific file version."""
    # Step 1: Get the storage location (S3/OSS signed URL)
    url  = f"{BASE_URL}/data/v1/projects/{project_id}/versions/{version_id}"
    data = client.get(url)
    download_url = (data.get("relationships", {})
                        .get("storage", {})
                        .get("meta", {})
                        .get("link", {})
                        .get("href"))

    if not download_url:
        print("No direct download URL. File may require OSS signed URL.")
        return False

    # Step 2: Stream download
    resp = requests.get(download_url, headers=client.headers(), stream=True)
    resp.raise_for_status()
    os.makedirs(os.path.dirname(output_path), exist_ok=True)
    with open(output_path, "wb") as f:
        for chunk in resp.iter_content(chunk_size=8192):
            f.write(chunk)
    print(f"Downloaded: {output_path}")
    return True

# Example usage:
# versions = get_versions(client, PROJECT_ID, ITEM_ID)
# for v in versions:
#     print(f"v{v['version_number']} — {v['last_modified']} by {v['created_by']}")
```

---

## Script 3: Upload a file to an ACC Docs folder

Uploading to ACC requires three steps: create a storage object, upload binary data, then create a new item/version.

```python
import os, requests, math

BASE_URL = "https://developer.api.autodesk.com"

def upload_file_to_acc(client, project_id, folder_id, local_file_path):
    """Upload a file to an ACC Docs folder. Returns the new item ID."""
    file_name = os.path.basename(local_file_path)
    file_size = os.path.getsize(local_file_path)

    # ── Step 1: Create a storage location (OSS object) ───────────────────────
    storage_resp = client.post(
        f"{BASE_URL}/data/v1/projects/{project_id}/storage",
        {
            "jsonapi": {"version": "1.0"},
            "data": {
                "type": "objects",
                "attributes": {"name": file_name},
                "relationships": {
                    "target": {
                        "data": {"type": "folders", "id": folder_id}
                    }
                }
            }
        }
    )
    object_id  = storage_resp["data"]["id"]
    upload_url = storage_resp["data"]["relationships"]["storage"]["meta"]["link"]["href"]

    # ── Step 2: Upload the binary to OSS ─────────────────────────────────────
    # For files >5 MB, use resumable upload. For simplicity, single-part shown here.
    with open(local_file_path, "rb") as f:
        file_data = f.read()

    upload_resp = requests.put(
        upload_url,
        headers={"Authorization": "Bearer " + client.get_token(),
                 "Content-Type":  "application/octet-stream"},
        data=file_data
    )
    upload_resp.raise_for_status()
    print(f"Uploaded binary: {file_name} ({file_size} bytes)")

    # ── Step 3a: Create a new Item (first upload of this file) ───────────────
    create_resp = client.post(
        f"{BASE_URL}/data/v1/projects/{project_id}/items",
        {
            "jsonapi": {"version": "1.0"},
            "data": {
                "type": "items",
                "attributes": {
                    "displayName": file_name,
                    "extension": {
                        "type":    "items:autodesk.core:File",
                        "version": "1.0"
                    }
                },
                "relationships": {
                    "tip": {
                        "data": {"type": "versions", "id": "1"}
                    },
                    "parent": {
                        "data": {"type": "folders", "id": folder_id}
                    }
                }
            },
            "included": [{
                "type": "versions",
                "id":   "1",
                "attributes": {
                    "name": file_name,
                    "extension": {
                        "type":    "versions:autodesk.core:File",
                        "version": "1.0"
                    }
                },
                "relationships": {
                    "storage": {
                        "data": {"type": "objects", "id": object_id}
                    }
                }
            }]
        }
    )
    item_id = create_resp["data"]["id"]
    print(f"Item created in ACC: {item_id}")
    return item_id

# Usage:
# item_id = upload_file_to_acc(client, PROJECT_ID, FOLDER_ID, r"C:\Drawings\A-DR-001.pdf")
```

---

## Script 4: Create a folder structure

```python
BASE_URL = "https://developer.api.autodesk.com"

def create_folder(client, project_id, parent_folder_id, folder_name):
    """Create a subfolder under an existing folder. Returns new folder ID."""
    resp = client.post(
        f"{BASE_URL}/data/v1/projects/{project_id}/folders",
        {
            "jsonapi": {"version": "1.0"},
            "data": {
                "type": "folders",
                "attributes": {
                    "name": folder_name,
                    "extension": {
                        "type":    "folders:autodesk.core:Folder",
                        "version": "1.0"
                    }
                },
                "relationships": {
                    "parent": {
                        "data": {"type": "folders", "id": parent_folder_id}
                    }
                }
            }
        }
    )
    new_id = resp["data"]["id"]
    print(f"Created folder: {folder_name} → {new_id}")
    return new_id

def create_folder_tree(client, project_id, root_folder_id, tree):
    """
    Create a nested folder structure from a dict.
    tree = {
        "01 Architecture": {
            "Drawings": {},
            "Models": {}
        },
        "02 Structure": {}
    }
    """
    def recurse(parent_id, subtree):
        for name, children in subtree.items():
            new_id = create_folder(client, project_id, parent_id, name)
            if children:
                recurse(new_id, children)

    recurse(root_folder_id, tree)

# Example:
# STANDARD_STRUCTURE = {
#     "01 Architecture": {"Drawings": {}, "Models": {}, "Reports": {}},
#     "02 Structure":    {"Drawings": {}, "Models": {}},
#     "03 MEP":          {"Drawings": {}, "Models": {}},
#     "04 Coordination": {"Clash Reports": {}, "Issue Log": {}},
#     "99 Admin":        {"BEP": {}, "Contracts": {}},
# }
# create_folder_tree(client, PROJECT_ID, ROOT_FOLDER_ID, STANDARD_STRUCTURE)
```

---

## Transmittals, Reviews & Packages (ACC Docs API)

For **Transmittals** (formally issuing document sets):

```
GET  /construction/transmittals/v1/projects/{projectId}/transmittals
POST /construction/transmittals/v1/projects/{projectId}/transmittals
GET  /construction/transmittals/v1/projects/{projectId}/transmittals/{transmittalId}
```

For **Document Reviews**:

```
GET  /construction/reviews/v1/projects/{projectId}/reviews
POST /construction/reviews/v1/projects/{projectId}/reviews
```

For **Packages** (bundled document sets):

```
GET  /construction/packages/v1/projects/{projectId}/packages
POST /construction/packages/v1/projects/{projectId}/packages
```

All require: `Authorization: Bearer {3-legged-token}` and `x-ads-region: US` (or `EU`)

---

## Common ACC Docs patterns

| Task                          | Approach                                      |
|-------------------------------|-----------------------------------------------|
| List drawings on a sheet set  | Traverse top folders → "Project Files" → discipline folder |
| Find latest PDF of a drawing  | `GET /items/{id}/versions` → filter by `.pdf` extension |
| Create a transmittal          | POST to transmittals API with file version IDs |
| Bulk upload from local folder | Loop `upload_file_to_acc()` with `os.walk()`  |
| Mirror local folder to ACC    | Compare file names + sizes before uploading   |
