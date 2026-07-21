# xhost API Reference

This is the underlying HTTP API that the MCP tools wrap. For normal agent usage, prefer the `mcp__xhost__*` tools — they handle auth automatically via OAuth and do not require any user-facing token step. This reference is here for deep dives, debugging, or direct programmatic access.

Base URL: `https://api.xhostd.com`

All authenticated endpoints require the header: `Authorization: Bearer <token>` where `<token>` is either the user's OAuth-issued bearer (carried by the MCP server) or a 30-day unified credential minted via the `get_credentials` MCP tool (git + Postgres + platform API, full default scopes).

All error responses use the envelope: `{"error": {"code": "<code>", "message": "<message>"}}`

> **Signing up** happens in the browser via Google sign-in on first OAuth authorization. There is no signup API.

---

## GET /apps

List all apps owned by the authenticated user.

**Request body:** None

**Response (200):**
```json
{
  "apps": [
    {
      "id": "uuid",
      "name": "my-app",
      "repo_url": "https://git.<domain>/<username>/<app>.git",
      "template": "static",
      "created_at": "2025-01-15T10:30:00Z",
      "channels": [
        {
          "id": "uuid",
          "name": "prod",
          "hostname": "my-app-alice.xhostd.com",
          "git_ref_binding": "branch:master",
          "current_sha": "abc1234567890abcdef1234567890abcdef12345",
          "status": "running"
        }
      ]
    }
  ]
}
```

---

## POST /apps

Create a new app. Provisions a git repository and a `prod` channel automatically.

**Required scope:** `repo:*`

**Request body:**
```json
{
  "name": "my-app",
  "template": "static"
}
```

- `name` (string, required) — Must be a valid DNS label and must not use a reserved prefix (see Hostname Rules)
- `template` (string, optional, default `"static"`) — Runtime template. Valid values: `"static"` (nginx static file serving), `"app"` (user-provided `install.sh` + `launch.sh`), and `"docker"`. The `app` template runs inside an `xhost-runtime` image with Node 22, Python 3.12, and build tools pre-installed. The user provides `install.sh` (optional, installs dependencies) and `launch.sh` (required, starts the app on `$PORT`). The `docker` template builds the `Dockerfile` at the repo root on every deploy and runs the image with its own `ENTRYPOINT`/`CMD`: the container must listen on `$PORT` (injected) and answer the health check `GET /` with a 2xx. Env vars are injected at run time only — never as build args, so secrets are unavailable during the build and must never be baked into the image. Charged image size (total minus warm-base layers) is capped per plan: free 512 MiB / basic 2 GiB / pro 4 GiB. Warm base images are exempt from the charged size: `node:22-slim`, `node:24-slim`, `python:3.11-slim`, `python:3.12-slim`, `python:3.13-slim`, `debian:trixie-slim`. Docker deploys stream `[build] ...` lines (queue position, build duration, image size vs cap) into the deploy log.

**Response (200):**
```json
{
  "id": "uuid",
  "name": "my-app",
  "repo_url": "https://git.<domain>/<username>/<app>.git",
  "template": "static",
  "created_at": "2025-01-15T10:30:00Z",
  "channels": [
    {
      "id": "uuid",
      "name": "prod",
      "hostname": "my-app-alice.xhostd.com",
      "git_ref_binding": "branch:master",
      "current_sha": null,
      "status": "provisioning"
    }
  ]
}
```

**Errors:**
- `bad_request` (400) — invalid app name, reserved prefix, name already taken, or invalid template
- `bad_request` (400) — could not provision git backend

---

## GET /apps/{app_id}

Get details of a single app by UUID.

**Response (200):** Same shape as a single entry in the `GET /apps` response (an App object with channels).

**Errors:**
- `not_found` (404) — app not found or not owned by caller

---

## DELETE /apps/{app_id}

Delete an app and all its channels. Tears down containers, networks, and Caddy routes.

**Response:** 204 No Content

**Errors:**
- `not_found` (404) — app not found or not owned by caller

---

## GET /apps/{app_id}/channels

List all channels for an app.

**Response (200):**
```json
[
  {
    "id": "uuid",
    "name": "prod",
    "hostname": "my-app-alice.xhostd.com",
    "git_ref_binding": "branch:master",
    "current_sha": "abc1234...",
    "status": "running"
  },
  {
    "id": "uuid",
    "name": "wildcard",
    "hostname": "wildcard-my-app-alice.xhostd.com",
    "git_ref_binding": "branch:*",
    "current_sha": null,
    "status": "provisioning"
  }
]
```

**Errors:**
- `not_found` (404) — app not found or not owned by caller

---

## POST /apps/{app_id}/channels

Create a new channel on an app.

**Required scope:** `channel:*`

**Request body:**
```json
{
  "name": "wildcard",
  "git_ref_binding": "branch:*"
}
```

- `name` (string, required) — Must be a valid DNS label. Cannot be `prod` (reserved; system-created).
- `git_ref_binding` (string, required) — Must match the pattern `branch:<name>` or `branch:*`. The `<name>` portion must match `^[A-Za-z0-9][A-Za-z0-9/_\-\.]*$`.

**Response (200):**
```json
{
  "id": "uuid",
  "name": "wildcard",
  "hostname": "wildcard-my-app-alice.xhostd.com",
  "git_ref_binding": "branch:*",
  "current_sha": null,
  "status": "provisioning"
}
```

**Errors:**
- `bad_request` (400) — invalid name, reserved channel name, or invalid `git_ref_binding` format

---

## GET /apps/{app_id}/channels/{channel_id}

Get details of a single channel.

**Response (200):** Same shape as a single entry in the `GET /apps/{app_id}/channels` response.

**Errors:**
- `not_found` (404) — app or channel not found

---

## DELETE /apps/{app_id}/channels/{channel_id}

Delete a channel. Cannot delete the `prod` channel.

**Response:** 204 No Content

**Errors:**
- `not_found` (404) — app or channel not found
- `bad_request` (400) — cannot delete the prod channel

---

## POST /apps/{app_id}/channels/{channel_id}/deploy

Manually enqueue a deploy of a specific SHA to a channel.

**Required scope:** `deploy:*`

**Request body:**
```json
{
  "sha": "abc1234567890abcdef1234567890abcdef12345"
}
```

- `sha` (string, optional) — A 40-character hex SHA or a branch name matching `^[A-Za-z0-9][A-Za-z0-9/_\-\.]*$`
- `ref` (string, optional) — A branch name to resolve and deploy; equivalent to passing a branch name as `sha`.

At least one of `sha` or `ref` must be provided. If both are given, `sha` wins and `ref` is ignored.

**Response (200):**
```json
{
  "deploy_id": "uuid",
  "channel_id": "uuid",
  "status": "queued"
}
```

**Note:** This endpoint must be called explicitly after `git push` to trigger a deploy. Pushing code does not automatically deploy.

**Errors:**
- `bad_request` (400) — invalid SHA format, or neither `sha` nor `ref` given
- `not_found` (404) — app or channel not found

---

## GET /apps/{app_id}/channels/{channel_id}/logs?deploy={deploy_id}

Fetch the deploy log as plain text.

**Response (200):** Plain text (`text/plain`) containing the build/deploy log.

**Errors:**
- `not_found` (404) — deploy not found or log not available yet

---

## GET /apps/{app_id}/channels/{channel_id}/images

Live built-image inventory for one channel (docker-template apps), newest first.

**Request body:** None

**Response (200):**
```json
{
  "images": [
    {
      "tag": "xhost-app-...:abc1234",
      "sha": "abc1234567890abcdef1234567890abcdef12345",
      "size_bytes": 412000000,
      "charged_size_bytes": 180000000,
      "matched_base": "node:22-slim",
      "created": 1752000000,
      "current": true
    }
  ],
  "image_cap_bytes": 536870912
}
```

- `charged_size_bytes` — the size counted against the plan cap (total minus warm-base layers); `matched_base` names the exempt warm base, or `null` if none matched.
- `current` — whether the image corresponds to the channel's currently deployed SHA.
- `image_cap_bytes` — the owner plan's charged-image-size cap.
- `images` is `null` (never an error) when the channel's host agent is unreachable, so callers can always render the rest.

**Errors:**
- `not_found` (404) — app or channel not found or not accessible

---

## GET /apps/{app_id}/tree

List all files in the repo at the given ref. Lets a stateless agent see the current contents before editing.

**Query parameters:**

- `ref` (string, optional, default `master`) — Branch name or 40-char SHA.

**Response (200):**
```json
{
  "ref": "master",
  "sha": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
  "files": [
    {"path": "index.html", "kind": "blob", "size": 142},
    {"path": "static/style.css", "kind": "blob", "size": 87}
  ]
}
```

**Errors:**
- `not_found` (404) — app not found, or ref does not exist (e.g. empty repo with no commits yet)

---

## GET /apps/{app_id}/blob

Return the raw bytes of a single file at the given ref. Useful when an agent needs to read existing content before modifying it.

**Query parameters:**

- `ref` (string, optional, default `master`) — Branch name or 40-char SHA.
- `path` (string, required) — Repository-relative file path.

**Response (200):** Raw file bytes (`application/octet-stream`).

**Errors:**
- `not_found` (404) — app, ref, or file not found

---

## POST /apps/{app_id}/changeset

Apply a sparse changeset to the repo and create one real git commit on top of `ref`'s current HEAD (or as the initial commit on an empty branch). Designed for agents without shell access or local git — string values upsert files, `null` deletes, absent paths are unchanged.

**Required scope:** `repo:*`

**Request body:**
```json
{
  "ref": "master",
  "message": "agent: update headline",
  "changes": {
    "index.html": "<!doctype html><h1>hello</h1>",
    "static/style.css": "body { font-family: sans-serif; }",
    "old-page.html": null
  }
}
```

- `ref` (string, optional, default `"master"`) — Target branch. Must match `^[A-Za-z0-9][A-Za-z0-9/_\-\.]*$`. Created if it does not yet exist.
- `message` (string, required) — Commit message. Must be non-empty.
- `changes` (object, required) — Map of repo-relative path → string (upsert) or `null` (delete). Paths must be relative and must not contain `..` segments.

**Response (200):**
```json
{
  "sha": "def456abcdef456abcdef456abcdef456abcdef4"
}
```

**Note:** This endpoint creates a commit but does not deploy. Pass the returned `sha` to `POST /apps/{app_id}/channels/{channel_id}/deploy` to ship it. Same two-step model as `git push` followed by deploy.

**Errors:**
- `bad_request` (400) — invalid path, invalid branch name, empty message, or malformed changeset
- `not_found` (404) — app not found

---

## POST /apps/{app_id}/env

Set (upsert) an environment variable or secret on an app.

**Required scope:** `deploy:*`

**Request body:**
```json
{
  "key": "MY_VAR",
  "value": "my-value",
  "kind": "env",
  "channel_id": "uuid"
}
```

- `key` (string, required) — Must match `^[A-Z_][A-Z0-9_]*$`. Reserved keys (system-injected) are rejected: `XHOST_USER`, `XHOST_SHA`, `DATABASE_URL`, `DATABASE_HOST`, `DATABASE_PASSWORD`, `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_REGION`.
- `value` (string, required) — The value to set (stored encrypted).
- `kind` (string, optional, default `env`) — `env` (plain variable) or `secret`. Secret values are omitted from list responses (metadata only); the single reveal path is `GET /apps/{app_id}/env/{key}/value` (the web console's reveal uses the same endpoint), and each reveal is audit-logged.
- `channel_id` (string, optional) — Omit for an app-level default; set to a channel id for a per-channel override. At deploy time the channel override wins over the app default, and system-injected keys win over both.

**Response:** 204 No Content

**Errors:**
- `bad_request` (400) — invalid key format or reserved key
- `not_found` (404) — channel not found on this app

---

## DELETE /apps/{app_id}/env/{key}

Delete an environment variable from an app.

**Query parameters:**
- `channel_id` (optional) — With it, deletes only that channel's override; without it, deletes the app-level default.

**Response:** 204 No Content

---

## GET /apps/{app_id}/env

List an app's environment variables and secrets.

**Query parameters:**
- `channel_id` (optional) — Without it, raw rows (app-level and per-channel). With it, the resolved view for that channel: app defaults merged with the channel's overrides, the override winning.

**Response (200):**
```json
{
  "env": [
    {
      "key": "MY_VAR",
      "kind": "env",
      "scope": "app",
      "channel_id": null,
      "updated_at": "2025-01-16T10:30:00Z",
      "value": "my-value"
    },
    {
      "key": "API_TOKEN",
      "kind": "secret",
      "scope": "channel",
      "channel_id": "uuid",
      "updated_at": "2025-01-16T10:31:00Z",
      "value": null
    }
  ]
}
```

- `scope` — `app` (app-level default) or `channel` (channel override).
- `value` — Cleartext for `kind: "env"`; **always `null` for secrets** in list responses. The only read path for a secret's value is `GET /apps/{app_id}/env/{key}/value` (audit-logged); MCP has no reveal tool.

---

## GET /apps/{app_id}/env/{key}/value

Reveal one env value — the only read path for `kind: "secret"`. Every call records an `env.reveal` journal event (visible in `GET /apps/{app_id}/events`) before the value is returned. The web console's click-to-reveal uses this same endpoint.

**Required scope:** `deploy:*`

**Query parameters:**
- `channel_id` (optional) — With it, resolved semantics: that channel's override wins, falling back to the app-level row when the channel has no override. Without it, the app-level row only.

**Response (200):**
```json
{
  "key": "API_TOKEN",
  "kind": "secret",
  "scope": "channel",
  "value": "the-decrypted-value"
}
```

- `scope` — `app` or `channel`, the scope the value resolved from.

**Errors:**
- `not_found` (404) — key not set at the requested scope, or value unavailable

---

## GET /apps/{app_id}/channels/{channel_id}/deploys/{deploy_id}/env

Return the env snapshot recorded when a deploy started — what the app actually ran with.

**Response (200):**
```json
{
  "deploy_id": "uuid",
  "env": [
    {"key": "MY_VAR", "kind": "env", "source": "app", "value": "my-value"},
    {"key": "API_TOKEN", "kind": "secret", "source": "channel", "value": null}
  ],
  "system_keys": ["DATABASE_URL", "S3_BUCKET", "XHOST_SHA", "XHOST_USER"]
}
```

- `source` — `app` or `channel`, the scope the value resolved from.
- Secret values are masked (`null`); system-injected keys are listed by name only (their values are credentials and are not stored in the snapshot).

**Errors:**
- `not_found` (404) — deploy not found, or it predates env snapshots

---

## GET /apps/{app_id}/channels/{channel_id}/postgres/snapshots

List a channel's pre-deploy Postgres snapshots, newest first. A snapshot is taken automatically before every non-static deploy (unless the app's env sets `XHOST_DEPLOY_SKIP_DB_SNAPSHOT=1`).

**Request body:** None

**Response (200):**
```json
[
  {
    "snapshot_id": "uuid",
    "deploy_id": "uuid",
    "created_at": "2025-01-16T10:30:00Z",
    "size_bytes": 81920
  }
]
```

**Errors:**
- `not_found` (404) — app/channel not found or channel Postgres not provisioned

---

## POST /apps/{app_id}/channels/{channel_id}/postgres/restore

Roll the channel's schema back to a prior snapshot. Staged: the live schema is renamed aside, the snapshot restores into a fresh schema, and the renamed copy is only dropped after the restore succeeds — a failed restore loses nothing. Each restore is audit-logged.

**Request body:**
```json
{
  "confirm_schema_name": "ch_<channel-uuid-hex>",
  "snapshot_id": "uuid"
}
```

- `confirm_schema_name` (string, required) — Must exactly match the channel's schema name (from `GET .../postgres`); mismatches are rejected.
- `snapshot_id` (string, required) — A snapshot id from the snapshots list.

**Response (200):** The channel's updated Postgres status:
```json
{
  "db_name": "u_...",
  "schema_name": "ch_...",
  "role_name": "r_...",
  "status": "ready",
  "last_error": null,
  "connection_count": 0,
  "connection_limit": 20,
  "password_set": true,
  "storage_bytes": 81920,
  "external_enabled": false,
  "external_host": "db.xhostd.com",
  "external_database": "my-app"
}
```

**Errors:**
- `bad_request` (400) — `invalid_confirmation` (schema name mismatch)
- `permission_denied` (403) — `prod_restore_blocked`: restoring the `prod` channel requires the app env `XHOST_ALLOW_PROD_RESTORE=1`
- `not_found` (404) — snapshot not found, or its file is missing
- `conflict` (409) — `channel_busy` (a deploy is queued/running), or the account is mid-move
- `service_unavailable` (503) — `postgres_unavailable`

---

## POST /apps/{app_id}/channels/{channel_id}/blob/credentials

Return the S3-compatible credentials for the channel's object store — the only payload carrying the decrypted secret. Each read is audit-logged. Note this is a POST with no body.

**Required scope:** `blob:*`

**Response (200):**
```json
{
  "access_key_id": "...",
  "secret_access_key": "...",
  "endpoint": "https://s3.xhostd.com",
  "region": "us-east-1",
  "bucket": "my-app-alice-xhostd-com"
}
```

**Errors:**
- `not_found` (404) — app/channel not found or channel blob not provisioned
- `conflict` (409) — `blob_not_ready`
- `service_unavailable` (503) — `blob_unavailable`

---

## GET /apps/{app_id}/channels/{channel_id}/blob

Return the channel's object-store status and usage.

**Request body:** None

**Response (200):**
```json
{
  "status": "ready",
  "last_error": null,
  "usage_bytes": 1048576,
  "external_enabled": false,
  "virtual_bucket": "my-app-alice-xhostd-com",
  "virtual_endpoint": "https://s3.xhostd.com",
  "region": "us-east-1"
}
```

**Errors:**
- `not_found` (404) — app/channel not found or channel blob not provisioned
- `service_unavailable` (503) — `blob_unavailable`

---

## POST /apps/{app_id}/channels/{channel_id}/domains

Attach a custom domain to a channel (max 5 per channel; domains are globally unique across xhost). Idempotent for the same channel — re-POSTing the same domain returns the existing row with its token preserved. Returns the DNS records the user must create at their registrar.

**Required scope:** `deploy:*`

**Request body:**
```json
{
  "domain": "myapp.com"
}
```

**Response (201):**
```json
{
  "domain": "myapp.com",
  "status": "pending",
  "reason": null,
  "dns_records": {
    "txt_host": "_xhost.myapp.com",
    "txt_value": "xhost-verify-...",
    "cname_target": "my-app-alice.xhostd.com",
    "a_values": ["198.51.100.7"]
  },
  "created_at": "2025-01-16T10:30:00Z",
  "verified_at": null
}
```

- `dns_records` — Create the TXT record with `txt_value`, plus a routing record: `CNAME` → `cname_target` for subdomains, or `A` → `a_values` at an apex. `a_values` may be empty if platform IP resolution transiently failed.

**Errors:**
- `bad_request` (400) — `invalid domain: <reason>` (`is_ip_address`, `invalid_idna`, `too_long`, `invalid_label`, `platform_domain_forbidden`, `needs_dot`) or `domain_limit_reached`
- `conflict` (409) — `domain_taken` (another channel owns this domain)

---

## POST /apps/{app_id}/channels/{channel_id}/domains/{domain}/verify

Re-run the TXT + routing DNS check for an attached domain and update its status. Idempotent and retryable — DNS propagation takes minutes, so a `pending` status with a transient reason is normal at first.

**Required scope:** `deploy:*`

**Response (200):** Same shape as the attach response, with updated `status`, `reason`, and `verified_at`. `status` flips `pending → verified` on a passing check. `reason` uses a closed vocabulary: `txt_nxdomain`, `txt_token_mismatch`, `dns_not_pointing`, `domain_nxdomain` (the four definitive failures — the only ones that downgrade a verified domain), plus the transient `txt_lookup_failed`, `dns_lookup_failed`, `platform_ip_unknown`.

**Errors:**
- `not_found` (404) — app/channel/domain not found

---

## GET /apps/{app_id}/channels/{channel_id}/domains

List every custom domain attached to a channel.

**Response (200):**
```json
{
  "domains": [ ... ]
}
```

Each item has the same shape as the attach response.

---

## DELETE /apps/{app_id}/channels/{channel_id}/domains/{domain}

Detach a custom domain. Removes the routing and stops certificate renewals; the existing cert expires on its own.

**Required scope:** `deploy:*`

**Response (200):**
```json
{
  "ok": true
}
```

**Errors:**
- `not_found` (404) — app/channel/domain not found

---

## POST /apps/{app_id}/github/sync

For apps connected to a GitHub source: fetch the latest GitHub commits into the app's internal xhost mirror without deploying. Deploys auto-sync anyway; use this to refresh the mirror or surface sync errors on their own. Requires the admin role (or higher) on the app.

**Request body:** None

**Response (200):**
```json
{
  "connected": true,
  "remote_url": "git@github.com:alice/my-app.git",
  "public_key": "ssh-ed25519 ...",
  "connected_at": "2025-01-16T10:30:00Z",
  "last_synced_at": "2025-01-16T10:35:00Z",
  "last_sync_status": "ok",
  "last_sync_error": null,
  "last_sync_refs": {"master": "abc1234..."}
}
```

**Errors:**
- `not_found` (404) — app not found, or no GitHub mirror connected

---

## GET /apps/{app_id}/events

List the app's activity events (attributed audit trail of project mutations: member changes, deploys, env writes and reveals, database operations, git pushes/commits), newest first. No secret values appear.

**Query parameters:**
- `limit` (integer, optional, default 50) — Max events to return, clamped to 1–100.
- `before` (timestamp, optional) — Return only events created before this instant (pagination cursor).

**Response (200):**
```json
{
  "events": [
    {
      "id": "uuid",
      "actor_username": "alice",
      "action": "deploy.create",
      "target": "prod",
      "detail": {},
      "created_at": "2025-01-16T10:30:00Z"
    }
  ],
  "next_before": "2025-01-16T10:30:00Z"
}
```

- `next_before` — Pass as `before` to fetch the next page; `null` when there are no more events.

---

## POST /exports

Queue a downloadable export (takeout) of a channel's or app's data: deployed code, a Postgres schema dump, and (when small enough) the channel's object-storage files. Env variable keys are included as blank placeholders — secret values are never exported. One non-terminal export per user at a time.

**Request body:**
```json
{
  "scope": "channel",
  "app_id": "uuid",
  "channel_id": "uuid"
}
```

- `scope` (string, required) — `channel` (one channel) or `app` (every channel).
- `app_id` (string, required) — The app to export.
- `channel_id` (string, required for `scope: "channel"`) — Ignored for `scope: "app"`.

**Response (202):**
```json
{
  "id": "uuid",
  "scope": "channel",
  "app_id": "uuid",
  "channel_id": "uuid",
  "status": "queued",
  "detail": null,
  "progress_pct": 0,
  "size_bytes": null,
  "error": null,
  "blobs_included": false,
  "blobs_reason": null,
  "blob_object_count": null,
  "blob_bytes_estimate": null,
  "created_at": "2025-01-16T10:30:00Z",
  "finished_at": null,
  "expires_at": null
}
```

**Errors:**
- `bad_request` (400) — invalid scope, missing `channel_id`, or nothing deployed to export
- `conflict` (409) — an export is already in progress, or the account is mid-move

---

## GET /exports

List the authenticated user's exports, newest first.

**Response (200):** A JSON array of export objects, same shape as the `POST /exports` response.

---

## GET /exports/{export_id}

Poll an export's progress. Statuses: `queued`, `running`, `ready`, `failed`.

**Response (200):** Same shape as the `POST /exports` response, with `progress_pct`, `size_bytes`, `error`, `blobs_included`/`blobs_reason`, and `expires_at` filled in as the build advances.

**Errors:**
- `not_found` (404) — export not found or not owned by caller

---

## POST /exports/{export_id}/download-token

Mint a short-lived `exports:read` token for a ready, owned export. The download routes (`GET /exports/{export_id}/download` for the archive; `GET /exports/{export_id}/download/blobs` for the object-storage tarball, when included) accept only this token as a bearer.

**Response (200):**
```json
{
  "token": "xh_...",
  "download_url": "/exports/{export_id}/download",
  "blobs_download_url": "/exports/{export_id}/download/blobs",
  "expires_at": "2025-01-17T10:30:00Z",
  "blobs_included": true,
  "blobs_reason": null
}
```

**Errors:**
- `not_found` (404) — export not found, not owned by caller, or not ready

---

## POST /credentials

Mint a 30-day unified credential for the authenticated user. The returned token serves as your git password, Postgres password, and platform API bearer, and carries the full default scope set (`repo:*`, `deploy:*`, `channel:*`, `db:*`, `blob:*`).

**Request body (optional):**
```json
{
  "scopes": ["repo:*", "db:*"]
}
```
`scopes`, when supplied, must be a non-empty subset of the default set and mints a least-privilege credential. Omit the body (or the field) to get the full default scopes.

**Response (200):**
```json
{
  "token": "xh_...",
  "username": "alice",
  "expires_at": "2025-01-16T10:30:00Z",
  "scopes": ["repo:*", "deploy:*", "channel:*", "db:*", "blob:*"]
}
```

To push: set the remote with the token in the **password** field — `https://<username>:<token>@git.xhostd.com/<username>/<app>.git` (the per-app `repo_url` from `GET /apps/{app_id}` already has the right path), then `git push`. Any username works; the password is what git.xhostd.com checks. (git.xhostd.com also accepts the token via `Authorization: Bearer` — e.g. `git config http.extraHeader "Authorization: Bearer <token>"` — but native `git` uses the Basic-password path above.) The token is valid for 30 days; re-mint after expiry.

---

## POST /feedback

Submit free-text feedback to the xhost team about platform friction (many iterations, an unclear tool/error, a missing capability). Attributed to the authenticated user. Fire-and-forget.

**Request body:**
```json
{
  "message": "Deploy logs don't stream — had to poll get_deploy_log repeatedly.",
  "app_id": "uuid"
}
```

- `message` (string, required) — The feedback text. Must be non-empty after trimming; max 4000 characters.
- `app_id` (string, optional) — Id of the app being worked on, for context. An unknown or inaccessible id is silently dropped (stored as null); the feedback still lands.

**Response (200):**
```json
{
  "id": "uuid",
  "status": "new"
}
```

**Errors:**
- `bad_request` (400) — empty message or message longer than 4000 characters

---

## GET /api/user/stats

Return dashboard stats for the authenticated user (self-scoped). No admin privileges required.

**Request body:** None

**Response (200):**
```json
{
  "username": "alice",
  "user_id": "uuid-string",
  "platform": {
    "apps": 3,
    "channels": 5,
    "running_channels": 4,
    "deploys_last_hour": 1,
    "deploys_last_day": 7,
    "success_last_day": 6,
    "failed_last_day": 1
  },
  "resources": {
    "mem_current_mb": 45.2,
    "mem_limit_mb": 256.0,
    "mem_percent": 17.7,
    "cpu_current_percent": 2.5
  },
  "sites": [
    {
      "hostname": "myapp-alice.xhostd.com",
      "repo": "alice/myapp",
      "branch": "master",
      "status": "running",
      "sha": "abc123def456"
    }
  ],
  "collected_at": "2025-01-15 10:30:00 UTC"
}
```

**Notes:**
- Returns only the calling user's own apps, channels, deploys, and resource usage.
- Resource usage (`resources`) reflects the user's systemd cgroup slice (memory and CPU budgets). Values are zero if cgroup limits are not configured.

---

## Hostname Rules

All user-facing names (app names, usernames, channel names) must be valid **DNS labels**:

- Lowercase letters (`a-z`), digits (`0-9`), and hyphens (`-`)
- 1 to 40 characters
- Cannot start or end with a hyphen
- Regex: `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`

**Reserved app name prefixes** (blocked as exact match or `<prefix>-*`):
`git`, `api`, `www`, `admin`, `preview`, `staging`

**Reserved channel names** (cannot be user-created):
`prod`

**Hostname derivation:**
- Production channel: `<app>-<username>.<domain>` (e.g., `myapp-alice.xhostd.com`)
- Other channels: `<channel>-<app>-<username>.<domain>` (e.g., `wildcard-myapp-alice.xhostd.com`)
- Fan-out preview channels: `preview-<branch-slug>-<app>-<username>.<domain>`

---

## Channel Status Values

- `provisioning` — Channel has been created but no deploy has completed yet
- `running` — A deploy has completed successfully and the channel is serving traffic

---

## Deploy Status Values

- `queued` — Deploy is waiting to be picked up by the worker
- `running` — Deploy is currently building/deploying
- `success` — Deploy completed successfully
- `failed` — Deploy failed (check logs for details)

---

## git_ref_binding Format

Must match the pattern `branch:<name>` or `branch:*`.

- `branch:master` — Triggers on pushes to the `master` branch
- `branch:staging` — Triggers on pushes to the `staging` branch
- `branch:*` — Wildcard; deploying a branch that matches the wildcard creates a child channel named `preview-<slug>` bound to `branch:<actual-branch-name>`.

The `<name>` portion (when not `*`) must match: `^[A-Za-z0-9][A-Za-z0-9/_\-\.]*$`

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `auth_required` | 401 | No Authorization header or invalid format |
| `token_invalid` | 401 | Token does not exist or is malformed |
| `token_revoked` | 401 | Token has been revoked |
| `scope_denied` | 403 | Token lacks the required scope |
| `permission_denied` | 403 | Admin privileges required |
| `admin_not_configured` | 403 | Server admin user not set up |
| `not_found` | 404 | Resource does not exist or is not owned by caller |
| `bad_request` | 400 | Validation failure (see message for details) |
| `conflict` | 409 | State conflict (e.g. `domain_taken`, `channel_busy`, export already running) |
| `bad_gateway` | 502 | Upstream service error |
| `service_unavailable` | 503 | Dependent service degraded (e.g. `postgres_unavailable`, `blob_unavailable`) |
| `internal_error` | 500 | Unexpected server error |

---

## Token Scopes

OAuth-issued bearer tokens (used by the MCP server) carry the full default scope set: `repo:*`, `deploy:*`, `channel:*`, `db:*`, `blob:*`.

Unified credentials minted via `POST /credentials` carry the full default scope set (`repo:*`, `deploy:*`, `channel:*`, `db:*`, `blob:*`) unless a narrower `scopes` subset is requested.

| Scope | Grants |
|-------|--------|
| `repo:*` | Create apps (POST /apps), push to git repos |
| `channel:*` | Create channels (POST /apps/{id}/channels) |
| `deploy:*` | Deploy (POST /apps/{id}/channels/{id}/deploy), manage env vars |
| `db:*` | Connect to Postgres through the database gateway (external DB access) |
| `blob:*` | Mint object-storage credentials for a channel |
