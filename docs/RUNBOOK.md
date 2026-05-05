# AHM Google Workspace MCP — Operator Runbook

This document is the operating manual for AHM's internal Google Workspace MCP deployment. It is written for whoever has to operate this system if the original builder is unreachable. Read the **Quick Reference** and **Access Prerequisites** sections first; everything else can be looked up as needed.

---

## Quick Reference

| Item | Value |
|---|---|
| GCP project display name | `claude-Google-workspace-mcp` |
| GCP project ID (use this in `gcloud`) | `glassy-landing-494919-q8` |
| GCP project number | `231621948533` |
| Cloud Run service name | `workspace-mcp` |
| Cloud Run region | `us-central1` |
| Cloud Run public URL | `https://workspace-mcp-231621948533.us-central1.run.app` |
| MCP endpoint (what users connect to) | `https://workspace-mcp-231621948533.us-central1.run.app/mcp` |
| GCS bucket (per-user encrypted OAuth tokens) | `gs://ahm-workspace-mcp-creds` |
| Cloud Run service account | `231621948533-compute@developer.gserviceaccount.com` |
| Secret Manager secrets | `google-oauth-client-id`, `google-oauth-client-secret`, `fastmcp-jwt-signing-key` |
| GitHub fork | `https://github.com/ckgAHM/google_workspace_mcp` |
| Upstream repo (parent) | `https://github.com/taylorwilsdon/google_workspace_mcp` |
| OAuth consent screen User Type | Internal (auto-allows all `@americanhatmakers.com` Workspace identities) |
| Currently shipped plugin version | v0.2.1 |
| User base | 5 users on the americanhatmakers.com Workspace domain |

---

## Access Prerequisites

Before you can operate this system, you need four kinds of access. Don't wait until something breaks to request them — propagating new IAM grants and admin invites takes hours-to-days, and incidents don't wait.

1. **GCP project Owner role on `glassy-landing-494919-q8`.** Grants you full admin: Cloud Run, Cloud Build, Artifact Registry, Secret Manager, IAM, GCS bucket. Without this you cannot redeploy, view logs, rotate secrets, or change IAM. Request from a current Owner: GCP Console → IAM → Grant Access → enter your email → role `Owner`.
2. **GitHub collaborator on `ckgAHM/google_workspace_mcp`** (or fork it under your own/org account). Required to commit code changes and pull upstream updates. If the original fork owner's account is inaccessible, fork the upstream `taylorwilsdon/google_workspace_mcp` again and re-apply our customizations from the docs in this repo.
3. **AHM Anthropic-org admin role.** Required to update the organization-level MCP connector and the organization plugin in Claude Desktop's admin settings. Request from a current org admin via the Anthropic Console.
4. **Google Workspace Super Admin** (americanhatmakers.com domain). Only matters if the OAuth consent screen needs reconfiguration or the GCP project itself ever needs to be re-linked to the Workspace org. Request from a current Workspace Super Admin in `admin.google.com`.

Plus, install on your local machine:

- **gcloud CLI** (`https://cloud.google.com/sdk/docs/install`). Authenticate with `gcloud auth login` and `gcloud auth application-default login`. Set project: `gcloud config set project glassy-landing-494919-q8`.
- **git** (`https://git-scm.com/download/win` for Windows or platform default elsewhere). Configure `user.name` and `user.email` to your AHM identity.
- **uv** (`https://astral.sh/uv`). Used by the MCP server build/run pipeline. PowerShell on Windows: `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`.
- **Python 3.11+** (uv will install it automatically when you run `uv sync` in the repo).

---

## Architecture Overview

### The whole stack in one paragraph

A Python FastMCP server hosted on Google Cloud Run exposes ~105 Google Workspace tools (Gmail, Drive, Calendar, Docs, Sheets, Slides, Forms, Chat, Tasks, Contacts, Custom Search) via an OAuth 2.1 + Dynamic Client Registration MCP endpoint. Claude Desktop users on the AHM americanhatmakers.com Workspace domain connect to the endpoint through a centrally-managed organization connector, sign in with their Google identity, and the server stores per-user encrypted refresh tokens in a Google Cloud Storage bucket. A separate Cowork plugin (also distributed at the AHM org level) provides workflow guidance and AHM-specific safety rules that govern how Claude uses those tools. Two layers of safety: server-side hard blocks (cannot be bypassed) for the most dangerous operations, and skill-level advisory rules (loaded into Claude's context) for operations where flexibility matters.

### Request flow when a user invokes a tool

```
User in Claude Desktop → AHM Cowork plugin loads relevant skill content into Claude's context
                       → Claude decides which tool to call → MCP tool invocation
                                                          → Org-managed Connector forwards to Cloud Run URL
                                                          → Cloud Run instance receives request
                                                          → Server reads user's encrypted token from GCS bucket
                                                          → Server makes Google API call on user's behalf
                                                          → Response flows back through the same path
```

### Server-side hard blocks (in code, can't be bypassed)

These live in the forked Python source and are enforced regardless of skill state. To remove or modify, you'd need to edit the source, redeploy, and have a deliberate reason.

- `gdrive/drive_tools.py`: `manage_drive_access` rejects `share_type="anyone"`; `set_drive_file_permissions` rejects `link_sharing` set to anything other than `"off"`. (Public Drive sharing is the only truly unrecoverable Drive operation.)
- `gcalendar/calendar_tools.py`: `manage_event`, `manage_out_of_office`, `manage_focus_time` all reject `action="delete"`. (Calendar deletes send cancellation emails to attendees with no undo.)
- `gcontacts/contacts_tools.py`: `manage_contact` and `manage_contacts_batch` reject `action="delete"`. (Google Contacts has no Trash; deletion is permanent.)
- `gmail/gmail_tools.py`: `manage_gmail_label` and `manage_gmail_filter` reject `action="delete"`. (Label deletion loses associations; filter deletion silently changes mail handling.)

Search for the comment `AHM safety block:` to find every instance.

### Skill-level safety rules (in plugin)

The Cowork plugin's `skills/managing-google-workspace/SKILL.md` contains a "AHM Safety Rules" section that Claude follows when the skill is loaded (which is most prompts touching Workspace data). Confirmation patterns for: Drive trash, internal Drive sharing, Gmail send (prefer drafts), Gmail filter modifications, Calendar event modifications, find-and-replace previews, Sheet overwrites, Forms with existing responses. Skill behavior is also encoded as JSON evaluations in `skills/managing-google-workspace/evaluations/` for regression-testing if the skill is ever modified.

---

## Common Operations

### Redeploy after a code change

```powershell
cd "<path to your local clone of the fork>"
# make code changes
uv run main.py --help          # syntax check; should print usage with no traceback
git add <changed-files>
git commit -m "describe the change"
git push                       # to your fork
gcloud run deploy workspace-mcp --source . --region us-central1
```

The deploy takes 90 seconds to 5 minutes depending on whether dependencies changed. The final output line shows the new revision name. Existing user sessions are unaffected (Cloud Run rolls 100% of traffic over after the new revision is healthy). Watch the Cloud Run console for any deployment errors.

### Update the AHM Cowork plugin and re-publish

The plugin source lives under `skills/managing-google-workspace/` in this repo. To rebuild a new version:

1. Edit `skills/managing-google-workspace/SKILL.md` or any reference/evaluation file.
2. Bump version in `.claude-plugin/plugin.json` (e.g., 0.2.1 → 0.2.2).
3. Rebuild the `.plugin` archive (it's a zip of the `skills/managing-google-workspace/` parent folder plus `.claude-plugin/plugin.json` and `README.md`):

   ```bash
   cd <plugin-source-dir>   # the dir that has .claude-plugin/, skills/, README.md
   zip -r ahm-google-workspace-vX.Y.Z.plugin .
   ```

   On Windows PowerShell:
   ```powershell
   Compress-Archive -Path .\.claude-plugin, .\skills, .\README.md -DestinationPath .\ahm-google-workspace-vX.Y.Z.zip
   Rename-Item .\ahm-google-workspace-vX.Y.Z.zip .\ahm-google-workspace-vX.Y.Z.plugin
   ```

4. Upload the new `.plugin` file to the AHM Anthropic-organization plugin distribution in the Anthropic Console. All 5 users will receive the new version automatically.

**Important:** as of plugin v0.2.0 we learned Cowork validates `.plugin` files and rejects `.mcp.json` at the plugin root. Keep MCP connector configuration in the org connector settings, not bundled in the plugin.

### Rotate a secret

To rotate the Google OAuth client secret (the most likely rotation):

1. In GCP Console → APIs & Services → Credentials, find the Web application OAuth client. Click **Reset client secret** (this generates a new secret and invalidates the old one).
2. Update Secret Manager:

   ```powershell
   echo "<new-secret-value>" | gcloud secrets versions add google-oauth-client-secret --data-file=-
   ```

3. Cloud Run automatically picks up the latest version on the next instance start. Force a restart by triggering a no-op deploy:

   ```powershell
   gcloud run services update workspace-mcp --region us-central1 --update-env-vars "ROTATION_DATE=$(Get-Date -Format yyyy-MM-dd)"
   ```

4. Existing user sessions may need to re-authenticate. Notify the team.

To rotate the FastMCP JWT signing key (`fastmcp-jwt-signing-key`): same procedure, but **all users will need to re-authenticate** because their session tokens were signed with the old key. Coordinate timing.

### View live logs

```powershell
gcloud run services logs tail workspace-mcp --region us-central1
```

Or for a specific time window:

```powershell
gcloud run services logs read workspace-mcp --region us-central1 --limit 200
```

For richer filtering and search, use Cloud Logging in the GCP Console: `https://console.cloud.google.com/logs/query?project=glassy-landing-494919-q8`.

### Sync upstream changes from `taylorwilsdon/google_workspace_mcp`

Every quarter or when upstream releases security patches:

```bash
git remote add upstream https://github.com/taylorwilsdon/google_workspace_mcp   # one-time
git fetch upstream
git checkout main
git merge upstream/main                                                          # resolve conflicts
# Test locally: uv sync && uv run main.py --help
# Deploy: gcloud run deploy workspace-mcp --source . --region us-central1
```

**Conflict zones to watch (these are AHM customizations that upstream changes can collide with):**
- `gdrive/drive_tools.py` — `manage_drive_access` and `set_drive_file_permissions` AHM safety blocks
- `gcalendar/calendar_tools.py` — three `manage_*` delete blocks
- `gcontacts/contacts_tools.py` — two `manage_contact*` delete blocks
- `gmail/gmail_tools.py` — two `manage_gmail_*` delete blocks
- `main.py` — Apps Script removed from `SERVICE_MODULES` and emoji map
- `Dockerfile` — uses `--extra gcs` not `--extra disk`
- `.dockerignore` — has secrets exclusions

After merging, re-run the plugin's evaluation suite against the deployed server to confirm safety blocks still enforce.

### Add a new user

The flow is the same as for the existing 5 users:

1. The new user's email must be on the americanhatmakers.com Workspace domain. (Internal consent screen rejects all other domains automatically.)
2. The user opens Claude Desktop. The org-managed connector and plugin should auto-appear.
3. They click **Connect** on the Google Workspace (AHM) connector, sign in with their Google account, allow scopes.
4. Their encrypted refresh token is now stored in `gs://ahm-workspace-mcp-creds`.

No GCP IAM change needed for end users.

### Remove a user (revoke access)

Two-step revocation:

1. **Google side** (the actual access revocation): the user removes their consent at `https://myaccount.google.com/permissions` → finds the AHM Workspace MCP app → Remove access. This invalidates their refresh token immediately. From this moment, server-side calls on their behalf will fail.
2. **Optional cleanup of stored tokens**: delete the user's object from `gs://ahm-workspace-mcp-creds`. Object naming is per-email; use `gcloud storage ls gs://ahm-workspace-mcp-creds/` to find. (Not strictly required since the token is already invalidated, but tidy.)

If a user is offboarded by Google Workspace admin (account suspended), step 1 happens automatically when their identity is disabled — Google revokes all OAuth grants for suspended accounts.

---

## Troubleshooting

### A user reports "I can't connect" or "OAuth fails"

1. Confirm their email domain. Only `@americanhatmakers.com` works (Internal consent screen).
2. Confirm the org connector is showing as enabled in their Claude Desktop. Settings → Connectors → "Google Workspace (AHM)" should show; if not, the org connector configuration in the Anthropic Console may have been disabled or scoped to fewer users.
3. Check Cloud Run logs for their request: `gcloud run services logs read workspace-mcp --region us-central1 --limit 100`. Look for the user's email in OAuth callback log lines and any error context.
4. Have the user disconnect (Settings → Connectors → "..." → Disconnect) and reconnect to force a fresh OAuth round-trip.

### Tools aren't appearing in Claude Desktop

1. Have the user fully restart Claude Desktop (close from system tray, reopen).
2. Confirm the Cloud Run service is healthy: hit `https://workspace-mcp-231621948533.us-central1.run.app/.well-known/oauth-authorization-server` in a browser. You should see a JSON document with all 11 service scopes listed and URLs pointing to the Cloud Run domain (not localhost).
3. If the JSON shows `localhost` URLs, the `WORKSPACE_EXTERNAL_URL` env var was lost. Restore: `gcloud run services update workspace-mcp --region us-central1 --update-env-vars "WORKSPACE_EXTERNAL_URL=https://workspace-mcp-231621948533.us-central1.run.app"`.
4. If the metadata endpoint returns 5xx, check Cloud Run logs and the latest revision's status in the GCP Console. A bad revision can be rolled back with `gcloud run services update-traffic workspace-mcp --region us-central1 --to-revisions <previous-revision-name>=100`.

### Specific tool fails with an error like "Public sharing is disabled in this deployment for safety"

This is by design — it's the AHM safety block firing. The error message tells the user what to do instead (use specific user/group/domain shares, or use the Google Workspace UI directly). If a legitimate use case requires bypassing the block, the change has to be deliberate: edit `gdrive/drive_tools.py`, redeploy, and document why in the commit message.

### Costs are spiking

1. Check Cloud Run usage: `https://console.cloud.google.com/run/detail/us-central1/workspace-mcp/metrics`. Look at request count, container instance hours, CPU.
2. Check API quotas: `https://console.cloud.google.com/iam-admin/quotas?project=glassy-landing-494919-q8`. Workspace APIs have free quotas; spikes indicate either a runaway loop or a stolen token being used at scale.
3. Cap max-instances if needed: `gcloud run services update workspace-mcp --region us-central1 --max-instances 1` (drops parallelism temporarily).
4. If a stolen token is suspected, rotate the JWT signing key (forces all users to re-authenticate) and review GCS bucket logs for unauthorized object reads.

### Plugin or connector validation fails when uploading to org

1. The `.plugin` file must be a valid zip with a `.claude-plugin/plugin.json` manifest at the root. Validate locally:

   ```powershell
   Expand-Archive -Path .\ahm-google-workspace-vX.Y.Z.plugin -DestinationPath .\plugin-check -Force
   Get-Content .\plugin-check\.claude-plugin\plugin.json | ConvertFrom-Json
   ```

   The `name` field must be kebab-case. The `version` field must be valid semver.
2. **Do not include `.mcp.json` at the plugin root.** Cowork's validator rejects this. MCP connector configuration belongs in the Anthropic Console org connector settings, not bundled in the plugin.

---

## Recovery / Incident Response

### A user's OAuth token is suspected leaked or compromised

1. Have the user revoke at `https://myaccount.google.com/permissions` immediately. This invalidates the refresh token at Google's end.
2. Optionally delete their token blob from `gs://ahm-workspace-mcp-creds`.
3. If you can't reach the user fast enough, **rotate the FastMCP JWT signing key** — this invalidates every session token signed with the old key, forcing all 5 users (not just the compromised one) to re-authenticate. Use this nuclear option only if you can't isolate to one user.

### A whole-bucket compromise is suspected (someone got read access to `gs://ahm-workspace-mcp-creds`)

1. Have all 5 users revoke their Google grants immediately at `https://myaccount.google.com/permissions`.
2. Rotate both the OAuth client secret and the JWT signing key.
3. Audit Cloud Audit Logs on the bucket: `https://console.cloud.google.com/logs/query?project=glassy-landing-494919-q8` filter `resource.type="gcs_bucket" AND resource.labels.bucket_name="ahm-workspace-mcp-creds"`.
4. Review IAM on the bucket: `gcloud storage buckets get-iam-policy gs://ahm-workspace-mcp-creds`. Only the Cloud Run service account `231621948533-compute@developer.gserviceaccount.com` should have `roles/storage.objectAdmin`. Anyone else with read access is suspect.

### The Cloud Run service is fully down

1. Check service status in GCP Console: `https://console.cloud.google.com/run/detail/us-central1/workspace-mcp`.
2. Roll back to the previous revision if the most recent deploy is bad:

   ```powershell
   gcloud run revisions list --service=workspace-mcp --region=us-central1
   gcloud run services update-traffic workspace-mcp --region us-central1 --to-revisions <previous-revision>=100
   ```

3. Check Cloud Build for build failures: `https://console.cloud.google.com/cloud-build/builds?project=glassy-landing-494919-q8`.
4. If Cloud Run itself is down (rare), users will see connection errors. Status: `https://status.cloud.google.com/`.

---

## Maintenance Schedule

| Cadence | Task |
|---|---|
| Weekly (first month) | Tail Cloud Run logs (`gcloud run services logs tail workspace-mcp --region us-central1`) for 5 minutes during business hours to spot anomalies |
| Monthly | Review GCP billing for the project. Confirm under expected baseline (small team usage typically < $10/month) |
| Quarterly | Sync upstream changes from `taylorwilsdon/google_workspace_mcp`. Re-run plugin evaluations to confirm safety blocks still hold |
| Quarterly | Audit IAM on the GCP project — confirm no unauthorized service accounts or users have access |
| Quarterly | Audit IAM on the GCS bucket — only the Cloud Run service account should have write access |
| Annually | Rotate the FastMCP JWT signing key |
| Annually | Review the OAuth client secret and rotate if it's old |
| As needed | When a team member leaves, ensure their Google grant is revoked (usually automatic via Workspace Super Admin disabling their account) |

---

## Contacts

| Role | Name | Email |
|---|---|---|
| Original builder / primary admin | Christopher | chris@americanhatmakers.com |
| Technical backup operator | Ori | ori@americanhatmakers.com |
| Anthropic support (Team/Enterprise plan) | Anthropic Console | `support@anthropic.com` or via Console |
| Google Cloud support | (Console) | `https://console.cloud.google.com/support` |
| Upstream MCP maintainer | Taylor Wilsdon | Issues at `https://github.com/taylorwilsdon/google_workspace_mcp/issues` |

### Access matrix (as of 2026-05-04)

| Access type | Christopher | Ori |
|---|---|---|
| GCP project Owner on `glassy-landing-494919-q8` | ✓ | ✓ |
| AHM Anthropic-org admin | ✓ | ✓ |
| Google Workspace Super Admin (americanhatmakers.com) | ✓ | ✓ |
| GitHub collaborator on `ckgAHM/google_workspace_mcp` | ✓ (owner) | _NOT YET — needs to be added_ |

If Ori is operating without Christopher and needs to commit code changes, this gap matters. To resolve: Christopher adds Ori at `https://github.com/ckgAHM/google_workspace_mcp/settings/access` with write permission. Until then, Ori can read the repo but not push code changes — they'd have to fork it under their own GitHub account and redeploy from that fork (Cloud Run deploys via `gcloud run deploy --source .` from a local checkout, so any working clone is sufficient for redeployments — Ori just couldn't push customizations back to the canonical fork).

---

## Where this document lives

- **Authoritative copy**: `docs/RUNBOOK.md` in the GitHub fork at `https://github.com/ckgAHM/google_workspace_mcp/blob/main/docs/RUNBOOK.md`. Always read the version on `main` for current truth.
- **Local copy**: in your local clone of the fork at `<repo-root>/docs/RUNBOOK.md`.
- **Original builder's machine**: `C:\Users\TEXAS-001\OneDrive\Documents\Claude\Projects\Claude Google Workspace MCP\Google Coud\google_workspace_mcp\docs\RUNBOOK.md`.

When changes are made to the system, update this file and commit. The version on `main` is the operating manual everyone reads.
