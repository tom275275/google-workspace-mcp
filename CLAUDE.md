# Google Workspace MCP — Cloud Run Deployment

## Overview

This is a fork of [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp),
deployed as a remote MCP server on Google Cloud Run for personal use by tschreiter@gmail.com.

Local changes from upstream are limited to: `CLAUDE.md`, `cloudbuild.yaml`, `README.md`, and the icon addition in `core/server.py`.

## Live Service

- **URL**: https://google-workspace-mcp-557522843498.us-central1.run.app
- **Health check**: https://google-workspace-mcp-557522843498.us-central1.run.app/health
- **MCP endpoint**: https://google-workspace-mcp-557522843498.us-central1.run.app/mcp
- **Region**: us-central1 (within Cloud Run always-free tier)
- **Project**: tom-personal-tools
- **Version**: taylorwilsdon workspace-mcp v1.14.3

## MCP Client Configuration

### Claude Code

Register via CLI (user scope, persists across projects):

```bash
claude mcp add --transport http --scope user google-workspace https://google-workspace-mcp-557522843498.us-central1.run.app/mcp
```

This writes to `~/.claude.json` — the correct location for user-scoped MCP servers.

> **Note**: `~/.claude/mcp.json` does NOT work for registering MCP servers in Claude Code.
> Always use `claude mcp add` or edit `~/.claude.json` directly.

### Claude.ai

Add via Settings → Integrations → Add custom integration, using the MCP endpoint URL above.
First use will trigger a Google OAuth consent flow in the browser.

## Architecture

- **Runtime**: Python 3.11 on Cloud Run (managed, serverless)
- **Transport**: `streamable-http` (MCP over HTTP, suitable for remote clients)
- **Auth**: OAuth 2.1 with Web application OAuth client
- **Secrets**: Stored in Google Secret Manager, mounted as env vars at runtime
- **Scaling**: Scales to zero when idle; cold start ~2-3s
- **Token persistence**: OAuth credentials stored in GCS bucket (`tom-personal-tools-mcp-state`), mounted at `/mnt/mcp-state` — survives container restarts
- **CI/CD**: Cloud Build trigger on push to `main` auto-deploys

## Enabled Tools

Gmail and Calendar are intentionally disabled — deferred to claude.ai's first-party integrations.
Chat and Slides are also disabled (not needed).

| Service | Enabled | Notes |
|---|---|---|
| Drive | ✅ | Full file management |
| Docs | ✅ | Create/read/edit documents |
| Sheets | ✅ | Read/write spreadsheets |
| Forms | ✅ | Form management |
| Tasks | ✅ | Task management |
| Contacts | ✅ | People API |
| Apps Script | ✅ | Automation |
| Search | ✅ | Programmable Search Engine |
| Gmail | ❌ | Use claude.ai Gmail integration |
| Calendar | ❌ | Use claude.ai Google Calendar integration |
| Chat | ❌ | Not needed |
| Slides | ❌ | Not needed |

## Environment Variables

| Variable | Source | Purpose |
|---|---|---|
| `GOOGLE_OAUTH_CLIENT_ID` | Secret Manager | OAuth Web app client ID |
| `GOOGLE_OAUTH_CLIENT_SECRET` | Secret Manager | OAuth Web app client secret |
| `MCP_ENABLE_OAUTH21` | Env var (`true`) | Enables OAuth 2.1 for remote use |
| `USER_GOOGLE_EMAIL` | Env var | Default account: tschreiter@gmail.com |
| `TOOLS` | Env var | Space-separated list of enabled services |
| `WORKSPACE_MCP_BASE_URI` | Env var | Public Cloud Run base URL |
| `GOOGLE_OAUTH_REDIRECT_URI` | Env var | Full OAuth callback URL |
| `WORKSPACE_EXTERNAL_URL` | Env var | Overrides base_uri:port for OAuth metadata |
| `WORKSPACE_MCP_CREDENTIALS_DIR` | Env var (`/mnt/mcp-state/credentials`) | Points credential store at GCS-backed volume |

> **Critical**: `WORKSPACE_EXTERNAL_URL` must be set to the public Cloud Run URL (without port).
> Without it, the server advertises `:8000` in its OAuth metadata, breaking all auth flows.

## GCP Resources

- **Cloud Run service**: `google-workspace-mcp` (us-central1)
- **Artifact Registry repo**: `cloud-run-source-deploy` (us-central1)
- **Secret Manager secrets**: `GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`
- **GCS bucket**: `tom-personal-tools-mcp-state` (us-central1) — persists OAuth credential files across container restarts
- **OAuth client**: `google-workspace-mcp-cloudrun-web` (**Web application** type) in tom-personal-tools
- **Authorized redirect URI**: `https://google-workspace-mcp-557522843498.us-central1.run.app/oauth2callback`

> **Important**: OAuth client must be **Web application** type, not Desktop app.
> Desktop app clients only allow localhost redirect URIs and will block Cloud Run callbacks.

## Enabled Google APIs (tom-personal-tools)

All required APIs are enabled:
- Drive, Calendar, Docs, Sheets, Slides, Forms, Tasks, Gmail, Chat
- People API (`people.googleapis.com`) — for Contacts tools
- Apps Script API (`script.googleapis.com`) — for Apps Script tools
- Cloud Run, Cloud Build, Secret Manager, Artifact Registry

## Redeploying

Full redeploy command (also used by Cloud Build trigger on push to `main`):

```bash
cd /home/tom-unix/projects/google-workspace-mcp
gcloud run deploy google-workspace-mcp \
  --source . \
  --project=tom-personal-tools \
  --region=us-central1 \
  --platform=managed \
  --allow-unauthenticated \
  --port=8000 \
  --memory=512Mi \
  --set-secrets="GOOGLE_OAUTH_CLIENT_ID=GOOGLE_OAUTH_CLIENT_ID:latest,GOOGLE_OAUTH_CLIENT_SECRET=GOOGLE_OAUTH_CLIENT_SECRET:latest" \
  --set-env-vars="MCP_ENABLE_OAUTH21=true,USER_GOOGLE_EMAIL=tschreiter@gmail.com,TOOLS=drive docs sheets forms tasks contacts appscript search,WORKSPACE_MCP_BASE_URI=https://google-workspace-mcp-557522843498.us-central1.run.app,GOOGLE_OAUTH_REDIRECT_URI=https://google-workspace-mcp-557522843498.us-central1.run.app/oauth2callback,WORKSPACE_EXTERNAL_URL=https://google-workspace-mcp-557522843498.us-central1.run.app,WORKSPACE_MCP_CREDENTIALS_DIR=/mnt/mcp-state/credentials" \
  --add-volume=name=mcp-state,type=cloud-storage,bucket=tom-personal-tools-mcp-state \
  --add-volume-mount=volume=mcp-state,mount-path=/mnt/mcp-state \
  --timeout=300
```

## One-Time GCS Setup (already done)

Before the volume mount works, the bucket and IAM binding must exist. Run once:

```bash
# Create bucket (us-central1 = same region as Cloud Run, within free tier)
gcloud storage buckets create gs://tom-personal-tools-mcp-state \
  --project=tom-personal-tools \
  --location=us-central1 \
  --uniform-bucket-level-access

# Find the Cloud Run service account
gcloud run services describe google-workspace-mcp \
  --project=tom-personal-tools \
  --region=us-central1 \
  --format="value(spec.template.spec.serviceAccountName)"
# If blank, the default Compute SA is used:
# <project-number>-compute@developer.gserviceaccount.com

# Grant the service account write access to the bucket
gcloud storage buckets add-iam-policy-binding gs://tom-personal-tools-mcp-state \
  --member="serviceAccount:<SA_EMAIL>" \
  --role="roles/storage.objectAdmin"
```

## Pulling Upstream Updates

```bash
git fetch upstream
git log HEAD..upstream/main --oneline   # preview changes
git merge upstream/main
git push                                # triggers auto-deploy via Cloud Build
```

## Rotating OAuth Credentials

```bash
echo -n "NEW_CLIENT_ID" | gcloud secrets versions add GOOGLE_OAUTH_CLIENT_ID \
  --data-file=- --project=tom-personal-tools

echo -n "NEW_CLIENT_SECRET" | gcloud secrets versions add GOOGLE_OAUTH_CLIENT_SECRET \
  --data-file=- --project=tom-personal-tools
```

Redeploy after rotating to pick up the new secret versions.

## Lessons Learned

### OAuth client type matters
Desktop app OAuth clients only allow `localhost` redirect URIs — they cannot be used with
hosted servers. Always use **Web application** type for Cloud Run deployments, with the
`/oauth2callback` path registered as an authorized redirect URI. You cannot change the
type after creation; create a new client.

### WORKSPACE_EXTERNAL_URL is required
Cloud Run containers listen on port 8000 internally but expose HTTPS on port 443 publicly.
Without `WORKSPACE_EXTERNAL_URL`, the server computes its OAuth metadata URLs as
`https://...run.app:8000/...`, which is unreachable externally. This causes silent auth
failures in all MCP clients.

### MCP server registration in Claude Code
`~/.claude/mcp.json` is not read by Claude Code for MCP server registration — it's ignored.
Use `claude mcp add --transport http --scope user <name> <url>` which writes to `~/.claude.json`.

### gcloud auth in WSL
`gcloud auth login` requires `xdg-utils` to open a browser from WSL:
```bash
sudo apt install xdg-utils
echo 'export BROWSER="/mnt/c/Program Files (x86)/Google/Chrome/Application/chrome.exe"' >> ~/.bashrc
source ~/.bashrc
```

### Tool selection strategy
Don't blindly enable all tools. Claude Code already has first-party integrations for Gmail
and Calendar via claude.ai — enabling duplicate tools wastes context window space and
creates ambiguity about which tool to use. Audit existing integrations before enabling
services in a new MCP server.

### Cloud Run filesystem is ephemeral — use GCS for state
Each container restart (including scale-from-zero) wipes the local filesystem. OAuth credential
files stored locally are lost, requiring full re-auth on every cold start. Fix: mount a GCS
bucket via `--add-volume` / `--add-volume-mount` and point `WORKSPACE_MCP_CREDENTIALS_DIR` at
the mount path. The credential JSON then persists across restarts with zero code changes.

### Icon support
MCP spec 2025-11-25 added icon support via the `icons` array on the `Implementation` object.
FastMCP 3.1.1+ supports this via the `icons` parameter on the `FastMCP` constructor.
Use base64 data URIs for SVGs to avoid external URL dependencies:
```python
from mcp.types import Icon
FastMCP(icons=[Icon(src="data:image/svg+xml;base64,...", mimeType="image/svg+xml", sizes=["any"])])
```
