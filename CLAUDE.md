# Google Workspace MCP — Cloud Run Deployment

## Overview

This is a fork of [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp),
deployed as a remote MCP server on Google Cloud Run for personal use by tschreiter@gmail.com.

## Live Service

- **URL**: https://google-workspace-mcp-557522843498.us-central1.run.app
- **Health check**: https://google-workspace-mcp-557522843498.us-central1.run.app/health
- **Region**: us-central1 (within Cloud Run free tier)
- **Project**: tom-personal-tools
- **Version**: taylorwilsdon workspace-mcp v1.14.3

## MCP Client Configuration

Add this to your Claude Code `~/.claude/mcp.json`:

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

## Architecture

- **Runtime**: Python 3.11 on Cloud Run (managed, serverless)
- **Transport**: `streamable-http` (MCP over HTTP, suitable for remote clients)
- **Auth**: OAuth 2.1 with Desktop OAuth client (no redirect URI configuration needed)
- **Secrets**: Stored in Google Secret Manager, mounted as env vars at runtime
- **Scaling**: Scales to zero when idle; cold start ~2-3s

## Environment Variables

| Variable | Source | Purpose |
|---|---|---|
| `GOOGLE_OAUTH_CLIENT_ID` | Secret Manager | OAuth Desktop client ID |
| `GOOGLE_OAUTH_CLIENT_SECRET` | Secret Manager | OAuth Desktop client secret |
| `MCP_ENABLE_OAUTH21` | Env var (=`true`) | Enables OAuth 2.1 for remote use |
| `USER_GOOGLE_EMAIL` | Env var | Default account: tschreiter@gmail.com |

## GCP Resources

- **Cloud Run service**: `google-workspace-mcp` (us-central1)
- **Artifact Registry repo**: `cloud-run-source-deploy` (us-central1) — stores built container images
- **Secret Manager secrets**: `GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`
- **OAuth client**: `google-workspace-mcp-cloudrun` (Desktop app type) in tom-personal-tools

## Redeploying

To redeploy after pulling upstream changes:

```bash
cd /home/tom-unix/projects/google-workspace-mcp
git pull upstream main          # pull latest from taylorwilsdon
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

## Updating Secrets

To rotate OAuth credentials:

```bash
echo -n "NEW_VALUE" | gcloud secrets versions add GOOGLE_OAUTH_CLIENT_ID --data-file=- --project=tom-personal-tools
echo -n "NEW_VALUE" | gcloud secrets versions add GOOGLE_OAUTH_CLIENT_SECRET --data-file=- --project=tom-personal-tools
```

Then redeploy to pick up the new secret versions.

## Upstream Sync

This repo tracks taylorwilsdon upstream. To check for updates:

```bash
git fetch upstream
git log HEAD..upstream/main --oneline
```

## Enabled Google APIs (tom-personal-tools)

All required Workspace APIs are already enabled:
- Gmail, Drive, Calendar, Docs, Sheets, Slides, Forms, Tasks, Chat, Contacts
