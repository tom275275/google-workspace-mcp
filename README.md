# Google Workspace MCP Server — Personal Cloud Run Deployment

A personal remote MCP server providing full natural language control over Google Workspace,
hosted on Google Cloud Run. Based on [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp) (v1.14.3).

## Live Service

| | |
|---|---|
| **MCP Endpoint** | `https://google-workspace-mcp-557522843498.us-central1.run.app/mcp` |
| **Health Check** | `https://google-workspace-mcp-557522843498.us-central1.run.app/health` |
| **Transport** | Streamable HTTP (MCP over HTTP) |
| **Account** | tschreiter@gmail.com |

## Connecting to Claude Code

Add to `~/.claude/mcp.json`:

```json
{
  "mcpServers": {
    "google-workspace": {
      "type": "http",
      "url": "https://google-workspace-mcp-557522843498.us-central1.run.app/mcp"
    }
  }
}
```

Restart Claude Code to load the server.

## Enabled Google Workspace Tools

All major Google Workspace services are available:

| Service | Capabilities |
|---|---|
| **Gmail** | Search, read, send, draft, label, and manage email |
| **Google Calendar** | Create, update, delete events; check availability; RSVP |
| **Google Drive** | Search, upload, download, organize files and folders |
| **Google Docs** | Create, read, edit, and format documents |
| **Google Sheets** | Read and write cell ranges, manage spreadsheets |
| **Google Slides** | Create and edit presentations, extract content |
| **Google Forms** | Create forms, manage questions, read responses |
| **Google Tasks** | Manage task lists and individual tasks |
| **Google Chat** | Send messages, manage spaces |
| **Contacts** | Look up and manage contacts via People API |
| **Apps Script** | Execute and manage Google Apps Script projects |
| **Custom Search** | Programmable Search Engine integration (requires PSE key) |

## Infrastructure

| Resource | Details |
|---|---|
| **GCP Project** | tom-personal-tools |
| **Cloud Run Region** | us-central1 (within always-free tier) |
| **Runtime** | Python 3.11 on Cloud Run managed |
| **Auth** | OAuth 2.1, Desktop OAuth client |
| **Secrets** | Google Secret Manager (`GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`) |
| **CI/CD** | Cloud Build trigger on push to `main` |
| **Container Registry** | Artifact Registry (`cloud-run-source-deploy`, us-central1) |

## Redeploying Manually

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
  --set-env-vars="MCP_ENABLE_OAUTH21=true,USER_GOOGLE_EMAIL=tschreiter@gmail.com" \
  --timeout=300
```

Pushes to `main` trigger automatic redeployment via Cloud Build.

## Pulling Upstream Updates

```bash
git fetch upstream
git log HEAD..upstream/main --oneline   # preview changes
git merge upstream/main
git push
```

Cloud Build will automatically deploy the updated version.

## Rotating OAuth Credentials

```bash
echo -n "NEW_CLIENT_ID" | gcloud secrets versions add GOOGLE_OAUTH_CLIENT_ID \
  --data-file=- --project=tom-personal-tools

echo -n "NEW_CLIENT_SECRET" | gcloud secrets versions add GOOGLE_OAUTH_CLIENT_SECRET \
  --data-file=- --project=tom-personal-tools
```

Then redeploy to pick up the new versions.

## Upstream

This deployment tracks [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp).
Local changes are limited to `CLAUDE.md`, `cloudbuild.yaml`, and this `README.md`.

License: MIT (upstream)
