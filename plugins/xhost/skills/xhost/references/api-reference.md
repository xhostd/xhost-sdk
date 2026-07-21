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
- `template` (string, optional, default `"static"`) — Runtime template. Valid values: `"static"` (nginx static file serving), `"app"` (user-provided `install.sh` + `launch.sh`), and `"docker"` (the `Dockerfile` at the repo root is built and run; the container must listen on `$PORT`). The `app` template runs inside an `xhost-runtime` image with Node 22, Python 3.12, and build tools pre-installed. The user provides `install.sh` (optional, installs dependencies) and `launch.sh` (required, starts the app on `$PORT`).

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
- `kind` (string, optional, default `env`) — `env` (plain variable) or `secret`. Secret values are never returned by the API (metadata only); they can only be revealed in the web console, where each reveal is audit-logged.
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
- `value` — Cleartext for `kind: "env"`; **always `null` for secrets** — secret values are never returned via the API or MCP.

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
| `permission_denied` | 403 | Admin privileges required |
| `admin_not_configured` | 403 | Server admin user not set up |
| `not_found` | 404 | Resource does not exist or is not owned by caller |
| `bad_request` | 400 | Validation failure (see message for details) |
| `bad_gateway` | 502 | Upstream service error |
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
