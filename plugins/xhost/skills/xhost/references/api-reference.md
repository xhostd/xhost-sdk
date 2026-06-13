# xhost API Reference

This is the underlying HTTP API that the MCP tools wrap. For normal agent usage, prefer the `mcp__xhost__*` tools — they handle auth automatically via OAuth and do not require any user-facing token step. This reference is here for deep dives, debugging, or direct programmatic access.

Base URL: `https://api.xhostd.com`

All authenticated endpoints require the header: `Authorization: Bearer <token>` where `<token>` is either the user's OAuth-issued bearer (carried by the MCP server) or a short-lived git-push token minted via the `get_git_credentials` MCP tool (repo-scoped only).

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
- `template` (string, optional, default `"static"`) — Runtime template. Valid values: `"static"` (nginx static file serving) and `"app"` (user-provided `install.sh` + `launch.sh`). `"node"` is accepted as an alias for `"app"`. The `app` template runs inside an `xhost-runtime` image with Node 22, Python 3.12, and build tools pre-installed. The user provides `install.sh` (optional, installs dependencies) and `launch.sh` (required, starts the app on `$PORT`).

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

- `sha` (string, required) — A 40-character hex SHA or a branch name matching `^[A-Za-z0-9][A-Za-z0-9/_\-\.]*$`

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
- `bad_request` (400) — invalid SHA format
- `not_found` (404) — app or channel not found

---

## GET /apps/{app_id}/channels/{channel_id}/logs?deploy={deploy_id}

Fetch the deploy log as plain text.

**Response (200):** Plain text (`text/plain`) containing the build/deploy log.

**Errors:**
- `not_found` (404) — deploy not found or log not available yet

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

Set (upsert) an environment variable on an app.

**Required scope:** `deploy:*`

**Request body:**
```json
{
  "key": "MY_VAR",
  "value": "my-value"
}
```

- `key` (string, required) — Must match `^[A-Z_][A-Z0-9_]*$`. Reserved keys `XHOST_USER` and `XHOST_SHA` are rejected.
- `value` (string, required) — The value to set (stored encrypted).

**Response:** 204 No Content

**Errors:**
- `bad_request` (400) — invalid key format or reserved key

---

## DELETE /apps/{app_id}/env/{key}

Delete an environment variable from an app.

**Response:** 204 No Content

---

## POST /git-credentials

Mint short-lived (24h) git-push credentials for the authenticated user. The returned token is repo-scoped (`repo:*`) only — it cannot deploy, manage channels, env vars, or domains.

**Request body:** None

**Response (200):**
```json
{
  "token": "xh_...",
  "username": "alice",
  "expires_at": "2025-01-16T10:30:00Z"
}
```

To push: set the remote to `https://<token>@git.xhostd.com/<username>/<app>.git` (the per-app `repo_url` from `GET /apps/{app_id}` already has the right path), then `git push`. Re-mint after expiry.

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
    "cpu_current_percent": 2.5,
    "cpu_avg_percent": 1.1,
    "cpu_usage_sec": 142.5
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
| `scope_reserved` | 403 | Attempted to use the reserved `db:*` scope |
| `permission_denied` | 403 | Admin privileges required |
| `admin_not_configured` | 403 | Server admin user not set up |
| `not_found` | 404 | Resource does not exist or is not owned by caller |
| `bad_request` | 400 | Validation failure (see message for details) |
| `bad_gateway` | 502 | Upstream service error |
| `internal_error` | 500 | Unexpected server error |

---

## Token Scopes

OAuth-issued bearer tokens (used by the MCP server) carry the full default scope set: `repo:*`, `deploy:*`, `channel:*`.

Git-push tokens minted via `POST /git-credentials` carry only `repo:*` — enough for `git push`/`git clone`, not enough for deploys, env vars, or channel management.

| Scope | Grants |
|-------|--------|
| `repo:*` | Create apps (POST /apps), push to git repos |
| `channel:*` | Create channels (POST /apps/{id}/channels) |
| `deploy:*` | Deploy (POST /apps/{id}/channels/{id}/deploy), manage env vars |
| `db:*` | Reserved; cannot be used |
