---
name: xhost
description: >-
  Use when the user wants to deploy a website or app, host a static site, put
  something online, publish a page, create a preview URL, check deployment
  status, manage env vars or custom domains, push from a local git checkout,
  or mentions xhost in any way. Covers the full lifecycle: create app → push
  code → deploy → live HTTPS URL, plus channels, env, snapshots, domains,
  and Google sign-in.
---

# xhost — Agent-First Hosting

xhost is hosting designed for agents. You create an app, push its code to the git repo the app owns, and deploy it. Every app gets a production HTTPS URL; named channels give preview URLs. The same `mcp__xhost__*` tools are available on Claude Code and on claude.ai (via the connector), so the procedure below is identical in both contexts — except that a runtime with no shell has no git, and there the `commit_files` fallback stands in for the push.

## Authentication

Tools are already authenticated via OAuth — the plugin (Claude Code) and the connector (claude.ai) both handle this. Just call the tools.

If a tool reports unauthenticated:
- **Claude Code:** tell the user to run `/mcp`, select **xhost**, choose **Authenticate**. A browser opens, they sign in with Google (picking a username on first sign-in), approve, done.
- **claude.ai:** tell the user to reconnect the xhost connector in Settings → Connectors.

There is no API token to mint, copy, paste, or export. Do not ask the user for one.

If a tool listed in this skill or in llms-full.txt is missing from your runtime tool list, the client cached an older tool set at connect time. llms-full.txt is the source of truth — tell the user to reconnect (Claude Code: `/mcp` → xhost → reconnect; claude.ai: Settings → Connectors → reconnect xhost) to pick up the current tools.

## The golden path

Create the app, get its code into the repo, deploy. Names below are shown as `mcp__xhost__<name>` (Claude Code namespacing) but the underlying tool is the same on claude.ai — drop the prefix if the runtime exposes them unprefixed.

Read the recipe for the shape of app you build before step 1. Each recipe gives one complete app: every file, the exact calls, and the failure modes of that shape. The shape recipes are `references/guide-recipes-static.md`, `references/guide-recipes-app-node.md` (Express), `references/guide-recipes-app-python.md` (FastAPI) and `references/guide-recipes-docker.md`. `references/guide-index.md` lists every recipe.

1. **`mcp__xhost__create_app`** — args: `name`, `template` (`"static"` for plain HTML/CSS/JS, `"app"` for projects with `install.sh`/`launch.sh`, `"docker"` for projects with their own `Dockerfile` at the repo root). Returns the app object with `id`, `repo_url`, and `channels[0]` (the auto-created `prod` channel) including its `id` and `hostname`. Hold onto `app_id` and the prod `channel_id`.
2. **`git push` to the app's `repo_url`** — the standard path, whatever the size of the project. A push sends only the diff, so the second and every later edit is incremental, and it costs far fewer tokens than round-tripping whole file contents through a tool call. The five mechanical steps are in **Pushing code with git** below. **Pushing stores your code; it does not deploy.**
   - **Fallback, one case only:** when git is not available on the machine you are working on (a runtime with no shell), use **`mcp__xhost__commit_files`** — args: `app_id`, `message`, `files` (a `{path: content-or-null}` map; string upserts, null deletes), `ref` (default `"master"`). Returns `{sha}`. Send only files that are changing. On GitHub-connected apps this returns an error; push to GitHub instead.
3. **`mcp__xhost__deploy`** — args: `app_id`, `channel_id`, and either `ref` (a branch name, e.g. `"master"`; xhostd resolves it to that branch's current head — this is the form to use after a push) or `sha` (an exact commit — what `commit_files` returned). Returns `{deploy_id, channel_id, status: "queued"}`.

Then poll **`mcp__xhost__get_deploy_log`** with `app_id`, `channel_id`, `deploy_id` until the build finishes. For `static` apps deploys are seconds; for `app` template the first deploy runs `install.sh` and can take 30–90s. For `docker` the deploy builds the image first — the log streams `[build] ...` lines (queue position, build duration, image size vs your plan's cap).

To undo the last deploy, use **`mcp__xhost__rewind`** — args: `app_id`, `channel_id`. It's an instant one-step cutover to the previous deploy's image (no rebuild). To go back to an older commit, or to force a fresh rebuild, use `deploy` with that `sha` instead. Not available for `static` apps.

Naming rules: app and channel names are DNS labels — `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`, max 40 chars. Reserved app-name prefixes (rejected): `git`, `api`, `www`, `admin`, `preview`, `staging`. Channel name `prod` is reserved (auto-created).

## Pushing code with git

This is step 2 of the golden path, in full. It is the standard path for every project, whatever its size — first commit and hundredth alike. Full guide: `references/guide-git.md` (<https://docs.xhostd.com/guides/git>).

1. Call **`mcp__xhost__get_credentials`**. Returns `{token, username, expires_at, scopes}`. The token expires in 30 days and is the unified credential — one `xh_` secret carrying the full default scopes, so it is your git password, your Postgres password, and your platform API bearer at once.
2. Get the app's `repo_url` via `mcp__xhost__get_app` (`app_id`). It looks like `https://git.xhostd.com/<username>/<app>.git`.
3. Configure the remote with the token in the **password** field (any username works — the password is what git.xhostd.com checks):
   ```
   git remote add xhost "https://<username>:<token>@git.xhostd.com/<username>/<app>.git"
   ```
   (or `git remote set-url xhost ...` if it already exists). git.xhostd.com also accepts the token as an `Authorization: Bearer` header (`git config http.extraHeader "Authorization: Bearer <token>"`), but the password field is the normal path.
4. `git push xhost HEAD:master` (or `HEAD:<your-branch>`). xhost binds prod to `master`, but a fresh `git init` defaults to `main`. The explicit refspec pushes the current branch under the pinned name.
5. Trigger the build with **`mcp__xhost__deploy`** — pushing stores code but does not deploy. Pass `ref: "master"` (or the branch name) so xhostd resolves to HEAD; or pass an explicit `sha`.

The same token is your **Postgres password** when external database access is enabled in the console: `postgresql://<username>:<token>@db.xhostd.com:5432/<db>?sslmode=require` (`<db>` = app name for `prod`, else `<channel>-<app>`).

Rules: the token is short-lived; never commit it into the repo or write it into a file the user might check in. Re-mint by calling `get_credentials` again after expiry.

Where git genuinely is not available — a runtime with no shell, such as the claude.ai connector — use `commit_files` instead, then `deploy` the `sha` it returns. Do not reach for `sync_git` here: that tool only refreshes the mirror of a GitHub-connected app and is no part of this flow.

## Runtime contract — what makes a deploy succeed

A deploy is only marked **ready** if the app passes a **health check**, and there are **two ways to pass it** — whichever happens first within the time window:

1. **Answer HTTP.** The platform requests `/` on the app's port and accepts a **2xx**. A non-2xx (404/500/redirect-loop) fails.
2. **Create the readiness file.** Create the file whose path is in the injected `$XHOST_READY_FILE` env var. Nothing else about it matters — it can be empty; the platform only checks that it exists.

Use (1) for anything that serves HTTP. Use (2) when the app has no web surface at all — a queue consumer, a cron-style daemon, a stream processor — instead of adding a dummy HTTP listener just to satisfy the platform. Signal it once the app is genuinely doing its job (the consumer loop is subscribed and running), never as the first line of your start command. Such a channel still gets its hostname and URL; that URL just returns 502, which is expected.

If neither signal arrives in time the deploy fails regardless of whether the app "works." This is the most common reason a first deploy fails — design for it up front.

**`static`** — the committed files are served directly from the **repo root**. Put `index.html` at the root (it answers `/`). No build runs; commit the final HTML/CSS/JS, not un-built sources. Health window ~10s.

**`app`** — your process must:
- **Signal readiness one of the two ways.** If it serves HTTP: **listen on `0.0.0.0` and the injected `$XHOST_HTTP_PORT`** (read `$XHOST_HTTP_PORT` from the environment; never hardcode a port — frameworks that default to `localhost`/a fixed port, Flask `app.run()`, `next dev`, Vite preview, etc., will fail the check unless you pass the host and `$XHOST_HTTP_PORT` explicitly; `$PORT` is still injected at the same value, so existing apps keep working, but it is deprecated and will be removed — use `$XHOST_HTTP_PORT` in new code) and **return HTTP 200 at `/`** (a pure API whose routes live under `/api` 404s the health check even though it runs — add a minimal `/` handler that returns 200). If it does not serve HTTP: **create `$XHOST_READY_FILE`** once it is running, e.g. `open(os.environ["XHOST_READY_FILE"], "w").close()` in Python or `fs.closeSync(fs.openSync(process.env.XHOST_READY_FILE, "w"))` in Node — or `touch "$XHOST_READY_FILE"` from `launch.sh` if the process has no natural hook, placed as late as possible.
- **Boot within 120s.**
- **Stay within a small memory budget (~128 MB) at run time.** That cap applies to your running server, not to the build.
- **Run as a non-root user.** The container runs as `app`; the writable paths are `/app` (your code), `$HOME`, and `/tmp`. Writing anywhere else fails with `Permission denied`.
- **Put ALL installation in `install.sh`, never in `launch.sh`.** `install.sh` runs once at **build** time, as root, with a generous memory budget — that is the only place a system-wide install (`uv pip install`, `npm install -g`, `apt-get`) can succeed, and where a heavy build (a full Next.js build, a large `npm install`) belongs. `launch.sh` runs at boot as the non-root `app` user, so installing there fails on permissions and burns your 128 MB.

`install.sh` (optional) bakes dependencies into the image at build time; `launch.sh` (required) execs your long-running server at boot — both from the repo root. Minimal pair (Python):

```sh
# install.sh — runtime deps go here (build time, as root)
#!/bin/sh
set -e
# Prefer uv over pip — same packages, dramatically faster resolve + install.
uv pip install --system --no-cache flask gunicorn
```
```sh
# launch.sh — bind 0.0.0.0:$XHOST_HTTP_PORT and serve a 200 at "/"
#!/bin/sh
set -e
exec gunicorn --bind "0.0.0.0:$XHOST_HTTP_PORT" app:app
# node equivalent: exec node server.js  (server.js listens on process.env.XHOST_HTTP_PORT, host 0.0.0.0)
```

A worker with no HTTP surface uses the other signal instead — same file, no port:

```sh
# launch.sh — a queue consumer; readiness is the file, not a port
#!/bin/sh
set -e
exec python worker.py   # worker.py creates $XHOST_READY_FILE once its loop is running
```

**`docker`** — the repo has a `Dockerfile` at its **root**; xhost builds it on every deploy and runs the resulting image with pure Docker semantics — your image's own `ENTRYPOINT`/`CMD` runs (no `install.sh`/`launch.sh`). The contract:

- **Signal readiness one of the two ways** — **listen on `0.0.0.0` and the injected `$XHOST_HTTP_PORT`** and **return 2xx at `GET /`**, or **create `$XHOST_READY_FILE`** if the image has no HTTP surface. Same health check as `app`. The file signal needs no shell in the image: a distroless worker can just `open()` the path from its own code.
- **Env vars are injected at run time only, NEVER as build args.** Secrets are not available during the build and must never be baked into an image — read all config from the environment at startup.
- **Run migrations in the image's start command**, not at build time (the database is only reachable at run time):

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["sh", "-c", "alembic upgrade head && exec uvicorn app:app --host 0.0.0.0 --port $XHOST_HTTP_PORT"]
```

- **Charged image size is capped per plan**: basic 512 MiB / builder 2 GiB / indie 4 GiB / pro 12 GiB (the same caps apply to the `app` template). "Charged" = total image size minus warm-base layers; an over-cap build fails the deploy with the size, cap, and plan named in the log.
- **Prefer a warm base image** — `node:22-slim`, `node:24-slim`, `python:3.11-slim`, `python:3.12-slim`, `python:3.13-slim`, `debian:trixie-slim` — instant builds, base size exempt from your image cap.

When a deploy ends in `status: failed`, read `get_deploy_log`: `health check failed for container … no 2xx/3xx response at GET / … and no readiness file created at $XHOST_READY_FILE` means **neither** signal arrived in time — `/` didn't answer 200 on `$XHOST_HTTP_PORT` (wrong bind/port, no `/` route, slow boot) and no `$XHOST_READY_FILE` was created, or the app crashed before either could happen (a boot-time `Permission denied` — `launch.sh` runs as the non-root `app` user, so installing or writing outside `/app`, `$HOME`, `/tmp` crashes it).

When a deploy **succeeds but the app misbehaves later**, `get_deploy_log` is the wrong log — it only covers the build/boot window. Use **`mcp__xhost__get_runtime_log`** with `app_id` and `channel` (a name, e.g. `prod`) for the running app's stdout/stderr. Start with **no `command`**: the status header alone says whether the container is running, its exit code, whether it was OOM-killed (your app exceeded the plan's memory limit), and how many times it restarted. Then pass a `command` — the log is at `/log/app.log` (cwd `/log`) inside a throwaway container and your shell pipeline runs there, so `tail -n 200 app.log`, `grep -i error app.log | tail -20`, `awk`, `sed`, `wc -l` all work. There is no `jq`, `rg` or `less`, no network, and a 30 s limit. If a redeploy already replaced the container, the old one's log is archived — the header lists the readable `container_index` values, so you can still read why the previous version crashed. Only stdout/stderr is captured: an app that writes its logs to a *file* has nothing here.

## Channels (prod vs preview)

Every app has one `prod` channel bound to `branch:master`, created automatically. For preview/staging environments call **`mcp__xhost__create_channel`** with `app_id`, `name` (e.g. `staging`), `git_ref_binding` (`branch:<name>`, one explicit channel per branch — the legacy `branch:*` wildcard is rejected).

`deploy` targets a specific channel via `channel_id`. To find channel ids later: **`mcp__xhost__list_channels`** with `app_id`.

URL format:
- Prod: `https://<app>-<owner>.xhostd.com`
- Other channels: `https://<channel>-<app>-<owner>.xhostd.com`

`<owner>` is the user's xhost username (chosen at first sign-in). The exact `hostname` is always in the channel object returned by `create_app` / `list_channels` / `get_app` — read it from there rather than constructing it.

## Common follow-ups

- **Env vars & secrets:** `mcp__xhost__set_env` (`app_id`, `key`, `value`, optional `secret: true`, optional `channel` name) and `mcp__xhost__delete_env` (`app_id`, `key`, optional `channel`). Without `channel` the value is an app-level default; with it, a per-channel override that wins at deploy time. `mcp__xhost__list_env` (`app_id`, optional `channel`) lists entries — with `channel` it's the resolved view (`scope` = `app` or `channel`); plain values come back in cleartext, but **secret values are never returned via MCP** (metadata only — a secret can be revealed via the web console or the HTTP API `GET /apps/{app_id}/env/{key}/value`, and each reveal is audit-logged). `mcp__xhost__get_deploy_env` (`app_id`, `channel`, `deploy_id`) shows the env a past deploy ran with (secrets masked, system-injected keys by name only). Keys must match `^[A-Z_][A-Z0-9_]*$`. Reserved (don't try to set): `XHOST_USER`, `XHOST_SHA`, `XHOST_HTTP_PORT`, `PORT`, `XHOST_FORWARD_PORT`, `XHOST_READY_FILE`, `DATABASE_URL`, `DATABASE_HOST`, `DATABASE_PASSWORD`, `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_REGION`. Every channel automatically gets `DATABASE_URL` pointing at the channel's own Postgres database — read it from `process.env` (or equivalent); don't ask the user for a connection string.
- **Usage stats:** `mcp__xhost__get_app_stats` (`app_name`, optional `channel`, `window` ∈ `24h`/`7d`/`30d`).
- **Snapshots:** every non-static deploy auto-snapshots Postgres beforehand. `mcp__xhost__list_channel_snapshots` (`app_name`, `channel`) lists them newest-first; `mcp__xhost__restore_channel_db` (`app_name`, `channel`, `snapshot_id`) rolls the channel's database back to that snapshot. Refuses `prod` unless `XHOST_ALLOW_PROD_RESTORE=1` is set on the app.
- **Object storage (S3-compatible):** auto-provisioned per channel (like the database — no enable step), for unstructured blobs (uploads, generated assets, exports). `mcp__xhost__get_blob_credentials` (`app_name`, `channel`) returns the endpoint/bucket/key pair, and `mcp__xhost__get_blob_usage` (`app_name`, `channel`) reports bytes used. Deploys inject `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_REGION` — point any S3 SDK at those env vars rather than constructing them. Snapshot **restore** has no MCP tool but is available via the dashboard or the HTTP API (`POST .../blob/restore`); the **external-access** toggle is dashboard-only. Both are kept off MCP because a data rollback or making storage public is a deliberate human action.
- **Custom domains:** `mcp__xhost__add_custom_domain` (`app_name`, `channel`, `domain`) returns DNS instructions (TXT + CNAME or A) in the `instructions` field — relay that text to the user verbatim. After they create the records, call `mcp__xhost__verify_custom_domain` (same args). HTTPS is automatic once verified. `mcp__xhost__list_custom_domains` and `mcp__xhost__remove_custom_domain` are also available. Limit 5 per channel.

- **Public raw-TCP endpoints:** when the channel's HTTPS URL can't carry the protocol (a database protocol, a message broker, a game server, a custom binary protocol), `mcp__xhost__expose_port` (`app_name`, `channel`, optional `allow_cidrs`) returns a public `host:port` that forwards raw TCP into the container. `allow_cidrs` is a source-address allowlist (max 16 entries); omitting it means the whole internet can connect. The container side is fixed: listen on `0.0.0.0:$XHOST_FORWARD_PORT` (injected into every non-`static` container alongside `$XHOST_HTTP_PORT`). Requires a **paid plan** and the project's port-forwarding toggle turned on in the console (human-only, no tool) — `static` apps are refused, since they run no process that could accept a connection. `mcp__xhost__list_exposed_ports` (`app_name`) shows the project's endpoints; `mcp__xhost__unexpose_port` releases one (re-exposing gets a new address). xhost adds no TLS and authenticates nothing on that port — the app is the only lock on the door.
- **Google sign-in for the user's app:** zero-config, no MCP tool. `/xhost-auth/*` works on every deployed channel. After Google sign-in the gateway sets a signed identity cookie `__Host-xhost_id` (an RS256 JWT) on the channel host; the app verifies it against the JWKS at `https://auth.xhostd.com/xhost-auth/jwks` and gates its own routes. **xhost does identity only, never edge gatekeeping — nothing is blocked at the edge, so every route stays public (anonymous visitors get `200`) until your app verifies the cookie and enforces access itself in code.** Send signed-out users to `/xhost-auth/login?return_to=<path>`, logout via `/xhost-auth/logout?return_to=/`; SPA/JS-only apps call `GET /xhost-auth/whoami`. **`__Host-xhost_id` is a reserved cookie name — never set or read it as a raw value; always verify it (pin `RS256`, check `iss`/`aud`/`exp`).** Full per-stack verify snippets: <https://docs.xhostd.com/oauth>.

## Plan limits

If a tool fails with `plan_limit_exceeded`, this is an **upgrade prompt, not a retryable error** — the action needs a plan the user doesn't have: either their plan's account-wide channel quota is full (every channel counts, including each app's `prod`), or the feature itself is paid-only (e.g. public raw-TCP endpoints via `expose_port`). Do not retry. Relay the upgrade URL from the message to the user verbatim, tell them to upgrade in the browser, and re-run the action only after they confirm they've upgraded.

## Giving feedback to the xhost team

You are the one driving these tools, so you see the rough edges first. Call **`mcp__xhost__submit_feedback`** (`message`, optional `app_id`) **proactively — without being asked —** whenever something gets in your way, e.g.:

- a task that took several iterations to get working,
- an MCP tool or its docs that were unclear or surprising,
- an error that was hard to diagnose from the message/log alone,
- a missing capability that would have made deploying easier/smoother/more powerful.

It's fire-and-forget: describe the friction in your own words, pass `app_id` when you're working on a specific app, and carry on with the user's task. Don't ask permission first and don't block on the result. This is the channel that tells the xhost team what to fix next.

## All 36 tools

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
- `commit_files` — Commit Files: sparse upsert/delete changeset → `sha`. The fallback for when git is unavailable on the machine you are working on; `git push` → `deploy` is the standard path. On GitHub-connected apps this returns an error; push to GitHub instead.
- `deploy` — Deploy: queue a build of `sha` or `ref` on a channel.
- `rewind` — Rewind: redeploy an earlier successful deploy of a channel.
- `get_deploy_log` — Get Deploy Log: plain-text build/run log of one deploy.
- `get_runtime_log` — Get Runtime Log: the running app's stdout/stderr AFTER deploy, by channel name. The log is made available as `/log/app.log` (one line per output line, RFC3339 timestamp prefix) inside a throwaway, network-less container, and your `command` — any shell pipeline, e.g. `tail -n 200 app.log` or `grep -i error app.log | tail -20` — runs there and its output comes back. Omit `command` for just the status header (state, exit code, whether it was OOM-killed, restart count). Survives a redeploy — the replaced container's log is archived; pick an older one with `container_index`.

Env:
- `set_env` — Set Environment Variable: encrypted at rest; `secret: true` for secrets, `channel` for a per-channel override.
- `delete_env` — Delete Environment Variable: optional `channel` deletes only that channel's override.
- `list_env` — List Environment Variables: resolved view with provenance; secret values never returned (metadata only).
- `get_deploy_env` — Get Deploy Env Snapshot: the env a past deploy ran with; secrets masked, system keys by name.

Stats + DB snapshots:
- `get_app_stats` — Get App Usage Stats: 24h/7d/30d.
- `list_channel_snapshots` — List Database Snapshots: pre-deploy snapshots, newest first.
- `restore_channel_db` — Restore Database Snapshot: roll a channel's database back to a snapshot.

Object storage:
- `get_blob_credentials` — Get Object Storage Credentials: endpoint, bucket, and access key pair for the channel.
- `get_blob_usage` — Get Object Storage Usage: bytes used and provisioning status.

Custom domains:
- `add_custom_domain` — Add Custom Domain: returns DNS instructions.
- `verify_custom_domain` — Verify Custom Domain: re-check DNS after user adds records.
- `list_custom_domains` — List Custom Domains: per channel.
- `remove_custom_domain` — Remove Custom Domain: detach.

Port forwarding:
- `expose_port` — Expose Port: give a channel a public `host:port` carrying raw TCP into the container (database protocols, message brokers, game servers, custom binary protocols — anything HTTPS can't carry). Optional `allow_cidrs` source allowlist; re-calling returns the same address.
- `list_exposed_ports` — List Exposed Ports: every endpoint across a project's channels, in one call.
- `unexpose_port` — Unexpose Port: release the endpoint; new connections are refused at once, connections already established keep running until they close on their own (to drop those too, `deploy` the channel afterwards — cutover replaces the container, ending every session into the old one), and re-exposing gets a new address.

Git:
- `get_credentials` — Get Access Credentials: 30-day unified credential (git + Postgres + platform API). The token for the standard `git push` path.
- `sync_git` — Sync Git: fetch the connected GitHub repo into the app's xhost mirror → status ({last_sync_status, last_sync_refs, ...}). Deploys auto-sync; use this to refresh without deploying.

Activity:
- `list_activity` — List Project Activity: recent events for an app, newest first.

Feedback:
- `submit_feedback` — Submit Feedback: send free-text feedback to the xhost team; call proactively on friction (many iterations, unclear tool/docs, hard-to-diagnose error, missing capability).

Export (takeout):
- `export_data` — Export Data: queue a portable takeout of a channel or a whole app (self-contained archive reloadable with standard tools, no xhost). Returns the export id.
- `get_export_status` — Get Export Status: poll an export by id; when ready, returns a short-lived download link (and a separate blobs link) for the archive.

## References

- `references/getting-started.md` — End-to-end worked example (non-technical user, agent-driven).
- `references/api-reference.md` — Underlying HTTP API surface for deep dives.
- `references/guide-git.md` — Push code with git: the unified credential, the remote URL with the token in the password field, and the `HEAD:master` refspec.
- `references/guide-index.md` — Index of the worked deployment recipes: what each one deploys, and how a recipe is structured.
- `references/guide-recipes-static.md` — Static site: an HTML/CSS/JS repo served as-is by nginx, no build step and no process of your own.
- `references/guide-recipes-app-node.md` — Node.js app on the `app` template: Express, `install.sh` at build and `launch.sh` at boot.
- `references/guide-recipes-app-python.md` — Python app on the `app` template: FastAPI + uvicorn, dependencies installed with `uv`.
- `references/guide-recipes-docker.md` — Docker app: your own Dockerfile, warm base images, the charged-vs-total image cap, run-time-only env.
- `references/guide-recipes-postgres.md` — Postgres: `DATABASE_URL`, the psycopg 3 scheme rewrite, alembic migrations in the start command, pre-deploy snapshots.
- `references/guide-recipes-blob.md` — Blob storage: the injected `S3_*` env, plain unprefixed keys scoped per channel, uploading and serving objects with boto3.
- `references/guide-recipes-oauth.md` — Sign in with Google: verifying the `__Host-xhost_id` cookie, and why the platform gates no route for you.
- `references/guide-recipes-port-forwarding.md` — Raw TCP: a public `host:port` on `XHOST_FORWARD_PORT`, the console-only app toggle, and signalling readiness without serving HTTP.
- `references/guide-recipes-worker.md` — Background worker: a long-running loop that serves no HTTP, readiness by `$XHOST_READY_FILE`, and how to prove progress from the runtime log.
- `references/guide-recipes-commit-files.md` — Ship without git: the `commit_files` fallback for a runtime with no shell, sparse changesets, and `null` deletes.
- `references/guide-bkm.md` — Best-known methods: debugging with `get_runtime_log`, stack choices, upgrade-safe code, secrets, budgets, quota errors, and the undo path.

Every `references/guide-*.md` is a user-facing guide,
**generated** from `docs/guides/` by `scripts/build-docs.py`. Never hand-edit
one — edit the source and rebuild (design/DOCS.md).
