---
name: xhost
description: >-
  Use when the user wants to deploy a website or app, host a static site, put
  something online, publish a page, create a preview URL, check deployment
  status, manage env vars or custom domains, push from a local git checkout,
  or mentions xhost in any way. Covers the full lifecycle: create app → write
  files → deploy → live HTTPS URL, plus channels, env, snapshots, domains,
  and Google sign-in.
---

# xhost — Agent-First Hosting

xhost is hosting designed for agents. You create an app, write its files, and deploy it — all through MCP tools. Every app gets a production HTTPS URL; named channels give preview URLs. The same `mcp__xhost__*` tools are available on Claude Code and on claude.ai (via the connector), so the procedure below is identical in both contexts.

## Authentication

Tools are already authenticated via OAuth — the plugin (Claude Code) and the connector (claude.ai) both handle this. Just call the tools.

If a tool reports unauthenticated:
- **Claude Code:** tell the user to run `/mcp`, select **xhost**, choose **Authenticate**. A browser opens, they sign in with Google (picking a username on first sign-in), approve, done.
- **claude.ai:** tell the user to reconnect the xhost connector in Settings → Connectors.

There is no API token to mint, copy, paste, or export. Do not ask the user for one.

## The golden path

Three tools, in order. Names below are shown as `mcp__xhost__<name>` (Claude Code namespacing) but the underlying tool is the same on claude.ai — drop the prefix if the runtime exposes them unprefixed.

1. **`mcp__xhost__create_app`** — args: `name`, `template` (`"static"` for plain HTML/CSS/JS, `"app"` for projects with `install.sh`/`launch.sh`). Returns the app object with `id`, `repo_url`, and `channels[0]` (the auto-created `prod` channel) including its `id` and `hostname`. Hold onto `app_id` and the prod `channel_id`.
2. **`mcp__xhost__commit_files`** — args: `app_id`, `message`, `files` (a `{path: content-or-null}` map; string upserts, null deletes), `ref` (default `"master"`). Returns `{sha}`. Send only files that are changing.
3. **`mcp__xhost__deploy`** — args: `app_id`, `channel_id`, `sha` (from step 2). Returns `{deploy_id, channel_id, status: "queued"}`.

Then poll **`mcp__xhost__get_deploy_log`** with `app_id`, `channel_id`, `deploy_id` until the build finishes. For `static` apps deploys are seconds; for `app` template the first deploy runs `install.sh` and can take 30–90s.

Naming rules: app and channel names are DNS labels — `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`, max 40 chars. Reserved app-name prefixes (rejected): `git`, `api`, `www`, `admin`, `preview`, `staging`. Channel name `prod` is reserved (auto-created).

## Runtime contract — what makes a deploy succeed

A deploy is only marked **ready** if the app passes a **health check**: within a time window the platform requests `/` on the app's port and requires an HTTP **2xx** response. A non-2xx (404/500/redirect-loop), or nothing listening on `$PORT`, fails the deploy regardless of whether the app "works." This is the most common reason a first deploy fails — design for it up front.

**`static`** — the committed files are served directly from the **repo root**. Put `index.html` at the root (it answers `/`). No build runs; commit the final HTML/CSS/JS, not un-built sources. Health window ~10s.

**`app`** — your server must:
- **Listen on `0.0.0.0` and the injected `$PORT`.** Read `$PORT` from the environment; never hardcode a port. Frameworks that default to `localhost`/a fixed port (Flask `app.run()`, `next dev`, Vite preview, etc.) will fail the check — pass the host and `$PORT` explicitly.
- **Return HTTP 200 at `/`.** A pure API whose routes live under `/api` 404s the health check even though it runs — add a minimal `/` handler that returns 200.
- **Boot within 120s** of startup, including `install.sh`.
- **Stay within a small memory budget (~128 MB).** `install.sh` runs under the same budget every time the app starts, so keep installs and builds lean — heavy bundlers (a full Next.js build, a large `npm install`) can run out of memory and fail the deploy.

`install.sh` (optional) runs first to fetch dependencies, then `launch.sh` (required) execs your long-running server — both from the repo root. Minimal pair (Python):

```sh
# install.sh — runtime deps go here (runs before launch.sh)
#!/bin/sh
set -e
pip install flask gunicorn
```
```sh
# launch.sh — must bind 0.0.0.0:$PORT and serve a 200 at "/"
#!/bin/sh
set -e
exec gunicorn --bind "0.0.0.0:$PORT" app:app
# node equivalent: exec node server.js  (server.js listens on process.env.PORT, host 0.0.0.0)
```

When a deploy ends in `status: failed`, read `get_deploy_log`: `health check returned non-200` means `/` didn't answer 200 on `$PORT` in time (wrong bind/port, no `/` route, slow boot, or running out of memory during `install.sh`).

## Channels (prod vs preview)

Every app has one `prod` channel bound to `branch:master`, created automatically. For preview/staging environments call **`mcp__xhost__create_channel`** with `app_id`, `name` (e.g. `staging`), `git_ref_binding` (`branch:<name>`, one explicit channel per branch — the legacy `branch:*` wildcard is rejected).

`deploy` targets a specific channel via `channel_id`. To find channel ids later: **`mcp__xhost__list_channels`** with `app_id`.

URL format:
- Prod: `https://<app>-<owner>.xhostd.com`
- Other channels: `https://<channel>-<app>-<owner>.xhostd.com`

`<owner>` is the user's xhost username (chosen at first sign-in). The exact `hostname` is always in the channel object returned by `create_app` / `list_channels` / `get_app` — read it from there rather than constructing it.

## Common follow-ups

- **Env vars:** `mcp__xhost__set_env` (`app_id`, `key`, `value`) and `mcp__xhost__delete_env` (`app_id`, `key`). Keys must match `^[A-Z_][A-Z0-9_]*$`. Reserved (don't try to set): `XHOST_USER`, `XHOST_SHA`, `DATABASE_URL`, `DATABASE_HOST`, `DATABASE_PASSWORD`. Every channel automatically gets `DATABASE_URL` pointing at a per-channel Postgres schema — read it from `process.env` (or equivalent); don't ask the user for a connection string.
- **Usage stats:** `mcp__xhost__get_app_stats` (`app_name`, optional `channel`, `window` ∈ `24h`/`7d`/`30d`).
- **Snapshots:** every non-static deploy auto-snapshots Postgres beforehand. `mcp__xhost__list_channel_snapshots` (`app_name`, `channel`) lists them newest-first; `mcp__xhost__restore_channel_db` (`app_name`, `channel`, `snapshot_id`) rolls the channel's database back to that snapshot. Refuses `prod` unless `XHOST_ALLOW_PROD_RESTORE=1` is set on the app.
- **Custom domains:** `mcp__xhost__add_custom_domain` (`app_name`, `channel`, `domain`) returns DNS instructions (TXT + CNAME or A) in the `instructions` field — relay that text to the user verbatim. After they create the records, call `mcp__xhost__verify_custom_domain` (same args). HTTPS is automatic once verified. `mcp__xhost__list_custom_domains` and `mcp__xhost__remove_custom_domain` are also available. Limit 5 per channel.
- **Google sign-in for the user's app:** `mcp__xhost__set_oauth_paths` (`app_name`, `channel`, `paths`) protects URL prefixes (e.g. `["/admin/*"]`) with Google sign-in; the app receives the visitor's identity via `X-XHost-User-Email`/`-Name`/`-Sub` headers. Pass `paths: []` to disable. `mcp__xhost__get_oauth_paths` reads the current list.

## Working with git locally (Claude Code only)

If the user wants to push from a local working copy — e.g. iterating on a sizable project where `commit_files` round-trips through MCP would be slow — use git directly:

1. Call **`mcp__xhost__get_git_credentials`**. Returns `{token, username, expires_at}`. The token expires in 24 hours and is repo-scoped (cannot deploy, manage envs, or touch channels).
2. Get the app's `repo_url` via `mcp__xhost__get_app` (`app_id`). It looks like `https://git.xhostd.com/<username>/<app>.git`.
3. Configure the remote with the token inline:
   ```
   git remote add xhost "https://<token>@git.xhostd.com/<username>/<app>.git"
   ```
   (or `git remote set-url xhost ...` if it already exists).
4. `git push xhost master` (or your branch).
5. Trigger the build with **`mcp__xhost__deploy`** — pushing stores code but does not deploy. Pass `ref: "master"` (or the branch name) so xhostd resolves to HEAD; or pass an explicit `sha`.

Rules: the token is short-lived; never commit it into the repo or write it into a file the user might check in. Re-mint by calling `get_git_credentials` again after expiry. All non-git operations (deploys, envs, channels, domains, snapshots) still go through MCP tools — the git token cannot do them.

## All 24 tools

Apps:
- `list_apps` — List Apps: all apps owned by the user, with channels.
- `create_app` — Create App: provisions repo and `prod` channel.
- `get_app` — Get App Details: single app by id, including `repo_url`.
- `delete_app` — Delete App: tears down app + all channels + routes.

Channels:
- `list_channels` — List Channels: channel ids/hostnames for an app.
- `create_channel` — Create Channel: name + `branch:<name>` binding.
- `delete_channel` — Delete Channel: by `app_name`/`channel` name; refuses `prod`.

Files + deploy:
- `list_files` — List Repository Files: tree at a ref.
- `read_file` — Read File: single file contents at a ref.
- `commit_files` — Commit Files: sparse upsert/delete changeset → `sha`.
- `deploy` — Deploy: queue a build of `sha` or `ref` on a channel.
- `get_deploy_log` — Get Deploy Log: plain-text build/run log.

Env:
- `set_env` — Set Environment Variable: encrypted at rest.
- `delete_env` — Delete Environment Variable.

Stats + DB snapshots:
- `get_app_stats` — Get App Usage Stats: 24h/7d/30d.
- `list_channel_snapshots` — List Database Snapshots: pre-deploy snapshots, newest first.
- `restore_channel_db` — Restore Database Snapshot: roll a channel's database back to a snapshot.

Custom domains:
- `add_custom_domain` — Add Custom Domain: returns DNS instructions.
- `verify_custom_domain` — Verify Custom Domain: re-check DNS after user adds records.
- `list_custom_domains` — List Custom Domains: per channel.
- `remove_custom_domain` — Remove Custom Domain: detach.

Google sign-in (for the user's deployed app):
- `set_oauth_paths` — Configure Google Sign-In Paths: protect URL prefixes.
- `get_oauth_paths` — Get Google Sign-In Paths: current list.

Git:
- `get_git_credentials` — Get Git Push Credentials: 24h, repo-only scope, for local `git push`.

## References

- `references/getting-started.md` — End-to-end worked example (non-technical user, agent-driven).
- `references/api-reference.md` — Underlying HTTP API surface for deep dives.
