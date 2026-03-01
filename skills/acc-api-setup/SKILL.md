---
name: acc-api-setup
description: >
  Set up Python authentication and a reusable client for the Autodesk Platform Services
  (APS) API to access Autodesk Construction Cloud (ACC). Use this skill first before any
  other ACC API script — it handles OAuth2 token acquisition (2-legged and 3-legged),
  environment configuration, hub/project discovery, and the Python boilerplate that all
  ACC API scripts depend on. Trigger on phrases like "ACC API setup", "authenticate with
  APS", "get an APS token", "connect to ACC", "Autodesk API token", "APS credentials",
  "set up Forge API", "configure ACC API", "get access token Autodesk", or whenever a
  user is starting to write ACC/APS API scripts and needs the authentication foundation.
  Always offer this skill before any other ACC API skill if the user hasn't authenticated yet.
---

# ACC / APS API Setup & Authentication

Set up Python access to Autodesk Construction Cloud for:

**"$ARGUMENTS"**

## Prerequisites

Before writing any code, the user needs:
1. An APS app registered at [APS Developer Portal](https://aps.autodesk.com/)
2. A **Client ID** and **Client Secret** from their app
3. The app provisioned to their ACC account (ACC Admin → Apps & Integrations → Custom Integrations)
4. Python 3.8+ with `requests` installed (`pip install requests`)

Ask the user which flow they need:
- **2-legged** (server-to-server, no user login) — for automation scripts, reporting, data extraction
- **3-legged** (user authorises in browser) — for user-specific data: issues, RFIs, personal ACC content

---

## Script 1: 2-Legged Token (Client Credentials — most common for automation)

```python
import os
import requests
import base64
import time

# ── Configuration — use environment variables, never hardcode secrets ─────────
CLIENT_ID     = os.environ.get("APS_CLIENT_ID", "YOUR_CLIENT_ID")
CLIENT_SECRET = os.environ.get("APS_CLIENT_SECRET", "YOUR_CLIENT_SECRET")

# Scopes for typical ACC read/write operations
SCOPES = "data:read data:write account:read"

APS_AUTH_URL = "https://developer.api.autodesk.com/authentication/v2/token"
# ─────────────────────────────────────────────────────────────────────────────

class APSClient:
    """Reusable APS client that auto-refreshes the 2-legged token."""

    def __init__(self, client_id, client_secret, scopes):
        self.client_id     = client_id
        self.client_secret = client_secret
        self.scopes        = scopes
        self._token        = None
        self._expires_at   = 0

    def _credentials_header(self):
        """Base64-encode client_id:client_secret for Basic auth (OAuth2 v2)."""
        creds = base64.b64encode(
            (self.client_id + ":" + self.client_secret).encode()
        ).decode()
        return "Basic " + creds

    def get_token(self):
        """Return a valid access token, refreshing if needed."""
        if self._token and time.time() < self._expires_at - 60:
            return self._token

        resp = requests.post(
            APS_AUTH_URL,
            headers={
                "Authorization": self._credentials_header(),
                "Content-Type":  "application/x-www-form-urlencoded",
            },
            data={
                "grant_type": "client_credentials",
                "scope":      self.scopes,
            }
        )
        resp.raise_for_status()
        data = resp.json()
        self._token      = data["access_token"]
        self._expires_at = time.time() + data["expires_in"]
        return self._token

    def headers(self):
        """Return headers dict ready to pass to any APS API call."""
        return {
            "Authorization": "Bearer " + self.get_token(),
            "Content-Type":  "application/json",
        }

    def get(self, url, params=None):
        r = requests.get(url, headers=self.headers(), params=params)
        r.raise_for_status()
        return r.json()

    def post(self, url, payload):
        r = requests.post(url, headers=self.headers(), json=payload)
        r.raise_for_status()
        return r.json()

    def patch(self, url, payload):
        r = requests.patch(url, headers=self.headers(), json=payload)
        r.raise_for_status()
        return r.json()


# ── Usage example ─────────────────────────────────────────────────────────────
if __name__ == "__main__":
    client = APSClient(CLIENT_ID, CLIENT_SECRET, SCOPES)
    print("Token acquired:", client.get_token()[:20] + "...")
```

---

## Script 2: Discover your Hub ID and Project IDs

Run this after authentication to find the IDs you need for all other scripts.

```python
import requests, os
# Assumes APSClient class from Script 1 is available

BASE_URL = "https://developer.api.autodesk.com"

def list_hubs(client):
    """List all accessible hubs (ACC accounts)."""
    data = client.get(BASE_URL + "/project/v1/hubs")
    hubs = []
    for h in data.get("data", []):
        hubs.append({
            "Hub Name": h["attributes"]["name"],
            "Hub ID":   h["id"],           # Use this as HUB_ID in other scripts
            "Region":   h["attributes"].get("region", ""),
        })
    return hubs

def list_projects(client, hub_id):
    """List all projects in a hub."""
    data = client.get(BASE_URL + f"/project/v1/hubs/{hub_id}/projects")
    projects = []
    for p in data.get("data", []):
        # ACC project IDs start with "b." — strip it for Model Coordination API calls
        projects.append({
            "Project Name": p["attributes"]["name"],
            "Project ID":   p["id"],       # Use this (with b. prefix) for Data Management API
            "ACC ID":       p["id"].replace("b.", ""),  # Use this for Model Coord, Issues APIs
            "Status":       p["attributes"].get("status", ""),
        })
    return projects

if __name__ == "__main__":
    client = APSClient(CLIENT_ID, CLIENT_SECRET, SCOPES)

    print("=== HUBS ===")
    hubs = list_hubs(client)
    for h in hubs:
        print(f"  {h['Hub Name']}  →  Hub ID: {h['Hub ID']}")

    if hubs:
        hub_id = hubs[0]["Hub ID"]
        print(f"\n=== PROJECTS IN {hubs[0]['Hub Name']} ===")
        projects = list_projects(client, hub_id)
        for p in projects:
            print(f"  {p['Project Name']}")
            print(f"    Data Mgmt ID: {p['Project ID']}")
            print(f"    ACC ID:       {p['ACC ID']}")
```

---

## Script 3: 3-Legged Token (for user-scoped APIs — Issues, RFIs, Submittals)

Use this when the ACC API requires a user identity (not a service account).

```python
import os, webbrowser, urllib.parse, http.server, threading, requests, base64

CLIENT_ID     = os.environ.get("APS_CLIENT_ID")
CLIENT_SECRET = os.environ.get("APS_CLIENT_SECRET")
REDIRECT_URI  = "http://localhost:8080/callback"
SCOPES        = "data:read data:write"

# Step 1: Build the authorization URL and open it in the browser
AUTH_URL = "https://developer.api.autodesk.com/authentication/v2/authorize"
params = {
    "response_type": "code",
    "client_id":     CLIENT_ID,
    "redirect_uri":  REDIRECT_URI,
    "scope":         SCOPES,
}
auth_url = AUTH_URL + "?" + urllib.parse.urlencode(params)
print("Opening browser for authorization...")
webbrowser.open(auth_url)

# Step 2: Local server to catch the callback code
auth_code = {"value": None}

class CallbackHandler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
        auth_code["value"] = qs.get("code", [None])[0]
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"<h2>Auth complete. You can close this window.</h2>")
    def log_message(self, *args): pass

server = http.server.HTTPServer(("localhost", 8080), CallbackHandler)
t = threading.Thread(target=server.handle_request)
t.start()
t.join(timeout=60)

# Step 3: Exchange code for access token
creds = base64.b64encode((CLIENT_ID + ":" + CLIENT_SECRET).encode()).decode()
resp = requests.post(
    "https://developer.api.autodesk.com/authentication/v2/token",
    headers={"Authorization": "Basic " + creds,
             "Content-Type": "application/x-www-form-urlencoded"},
    data={"grant_type": "authorization_code",
          "code": auth_code["value"],
          "redirect_uri": REDIRECT_URI}
)
resp.raise_for_status()
token_data = resp.json()
ACCESS_TOKEN  = token_data["access_token"]
REFRESH_TOKEN = token_data["refresh_token"]
print("3-legged token acquired:", ACCESS_TOKEN[:20] + "...")
```

---

## Key API base URLs and ID conventions

| API                   | Base URL                                              | Auth     |
|-----------------------|-------------------------------------------------------|----------|
| Authentication v2     | `https://developer.api.autodesk.com/authentication/v2` | —        |
| Data Management v1    | `https://developer.api.autodesk.com/data/v1`          | 2-legged |
| Project v1 (hubs)     | `https://developer.api.autodesk.com/project/v1`       | 2-legged |
| ACC Issues v1         | `https://developer.api.autodesk.com/construction/issues/v1` | 3-legged |
| Model Coordination v2 | `https://developer.api.autodesk.com/modelcoordination/modelset/v2` | 2-legged |
| ACC Sheets v1         | `https://developer.api.autodesk.com/construction/sheets/v1` | 2-legged |

## ID convention guide

- **Hub ID**: `b.{account_id}` — used in `/project/v1/hubs/{hub_id}`
- **Project ID (Data Mgmt)**: `b.{project_id}` — used in `/data/v1/projects/{project_id}`
- **Project ID (Issues/Model Coord)**: `{project_id}` — **without** the `b.` prefix
- **Container ID** (Model Coordination): same as raw project ID without `b.`

## Environment variable setup

Always store credentials as environment variables — never hardcode:

```bash
# Windows (PowerShell)
$env:APS_CLIENT_ID     = "your_client_id"
$env:APS_CLIENT_SECRET = "your_client_secret"

# Mac/Linux
export APS_CLIENT_ID="your_client_id"
export APS_CLIENT_SECRET="your_client_secret"
```

Or use a `.env` file with `python-dotenv`:
```python
from dotenv import load_dotenv
load_dotenv()   # reads .env file automatically
```
