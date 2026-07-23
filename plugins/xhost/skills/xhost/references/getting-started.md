# Getting Started with xhost — Worked Example

This walks through what an agent does, end-to-end, when a non-technical user says something like *"build me a little site that lists my favourite coffee shops in Lisbon and put it online"*. No tokens, no copy-pasting, no shell.

## 0. Prerequisites

The xhost plugin is installed (Claude Code) or the xhost connector is enabled (claude.ai). The `mcp__xhost__*` tools are available. The user has authenticated once via OAuth — if not, point them to `/mcp` → xhost → Authenticate (Claude Code) or the connector's Connect button (claude.ai). They will sign in with Google and, on the first sign-in only, pick a **username** (lowercase letters, digits, hyphens; 1–40 chars; cannot start or end with a hyphen). That username becomes part of every public URL.

## 1. Decide on a name

Pick a DNS-label app name from the user's description — e.g. `lisbon-coffee`. Confirm the name with them in one line before creating anything. Rules: `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`, max 40 chars. Reserved prefixes (rejected by the API): `git`, `api`, `www`, `admin`, `preview`, `staging`.

## 2. Create the app

```
mcp__xhost__create_app(name="lisbon-coffee", template="static")
```

Use `template="static"` for plain HTML/CSS/JS (served directly). Use `template="app"` if the user wants a dynamic backend — that template runs `install.sh` then `launch.sh` in a runtime with Node 22, Python 3.13, and common build tools.

The response looks like:

```json
{
  "id": "f1e2…",
  "name": "lisbon-coffee",
  "repo_url": "https://git.xhostd.com/alice/lisbon-coffee.git",
  "template": "static",
  "channels": [
    {
      "id": "c0a1…",
      "name": "prod",
      "hostname": "lisbon-coffee-alice.xhostd.com",
      "git_ref_binding": "branch:master",
      "current_sha": null,
      "status": "provisioning"
    }
  ]
}
```

Remember `app_id` (`f1e2…`) and the prod channel's `id` (`c0a1…`).

## 3. Write the site

Build the file contents as strings. Then commit them all in one MCP call:

```
mcp__xhost__commit_files(
  app_id="f1e2…",
  message="initial site",
  files={
    "index.html": "<!doctype html><html><head><title>Lisbon Coffee</title>…",
    "style.css": "body{font-family:system-ui;max-width:40rem;margin:2rem auto}…",
  },
)
```

Returns `{"sha": "abc123…"}`. Only include files that are being added or changed; absent paths are left alone; pass `null` as a value to delete a file.

For an `app`-template project, the same call is used; include `install.sh` (optional) and `launch.sh` (required) in the changeset. The deploy only succeeds if the app passes a health check: within 120s the platform requests `/` on the app's port and needs an HTTP **200**. So the server must:

- **Bind `0.0.0.0` on `$PORT`** — read `$PORT` from the environment, don't hardcode. `python app.py` with Flask's default `app.run()` binds `localhost` on a fixed port and will fail the check; pass the host and `$PORT` explicitly.
- **Answer `/` with a 200** — an API whose routes are all under `/api` fails the check even though it runs; add a minimal `/` handler.
- **Boot within 120s and stay within a small memory budget (~128 MB).** `install.sh` runs under that budget every time the app starts, so keep it lean — a heavy build can run out of memory.

Example minimal pair (`install.sh` fetches deps, then `launch.sh` starts the server):

```sh
# install.sh
#!/bin/sh
set -e
pip install flask gunicorn
```
```sh
# launch.sh
#!/bin/sh
set -e
exec gunicorn --bind "0.0.0.0:$PORT" app:app
```

## 4. Deploy

```
mcp__xhost__deploy(
  app_id="f1e2…",
  channel_id="c0a1…",
  sha="abc123…",
)
```

Returns `{"deploy_id": "d…", "channel_id": "c0a1…", "status": "queued"}`. Deploys run async.

## 5. Watch the build

```
mcp__xhost__get_deploy_log(app_id="f1e2…", channel_id="c0a1…", deploy_id="d…")
```

Returns plain text. Poll until the log shows the build finished. `static` deploys take a few seconds; the first `app`-template deploy takes 30–90 seconds because `install.sh` runs.

## 6. Hand the user the URL

Read `hostname` from the prod channel (it was in step 2's response, and `mcp__xhost__list_channels` or `mcp__xhost__get_app` will return it later). Tell the user:

> Live at https://lisbon-coffee-alice.xhostd.com

## 7. Iterate

For each follow-up change — "add a section for tea shops", "fix the broken link" — call `commit_files` with the changed files, then `deploy` with the returned sha. The same `prod` channel id is reused.

To preview a change without touching production:

```
mcp__xhost__create_channel(app_id="f1e2…", name="draft", git_ref_binding="branch:draft")
mcp__xhost__commit_files(app_id="f1e2…", ref="draft", message="draft tea section", files={…})
mcp__xhost__deploy(app_id="f1e2…", channel_id="<draft channel id>", ref="draft")
```

The preview is live at `https://draft-lisbon-coffee-alice.xhostd.com`.

## 8. Optional extras

- **Env vars & secrets:** `mcp__xhost__set_env(app_id, key="STRIPE_KEY", value="sk_…", secret=True)` — mark credentials `secret=True` (values never readable back through MCP; console-only reveal, audit-logged), add `channel="staging"` for a per-channel override that wins over the app default at deploy time. `mcp__xhost__list_env(app_id, channel=…)` shows the resolved view. Every channel automatically has `DATABASE_URL` for its per-channel Postgres schema; don't set it.
- **Custom domain:** `mcp__xhost__add_custom_domain(app_name, channel, domain)` returns an `instructions` field with the exact TXT + CNAME/A records the user needs to add at their registrar — relay it verbatim. After they add the records, call `mcp__xhost__verify_custom_domain` with the same args. HTTPS works automatically once verified.
- **Google sign-in for parts of the site:** zero-config — no tool to call. `/xhost-auth/*` works on every channel. After sign-in the gateway sets a signed identity cookie `__Host-xhost_id` (an RS256 JWT) on the channel host; the app verifies it against the JWKS at `https://auth.xhostd.com/xhost-auth/jwks` (pin `RS256`, check `iss`/`aud`/`exp`) and gates its own routes. **xhost does identity only, never edge gatekeeping — nothing is blocked at the edge, so every route stays public (anonymous visitors get `200`) until your app verifies the cookie and enforces access itself in code.** Send signed-out users to `/xhost-auth/login?return_to=<path>`. `__Host-xhost_id` is a reserved cookie name. Per-stack verify snippets: <https://docs.xhostd.com/oauth>.
- **Local git workflow (Claude Code only):** if the user wants to push from a local checkout, call `mcp__xhost__get_credentials`, configure `git remote add xhost https://<username>:<token>@git.xhostd.com/<username>/<app>.git` (token in the password field — any username works), `git push`, then `mcp__xhost__deploy` with `ref="master"`. The token expires after 30 days; the same token is also your Postgres password and platform API bearer.

## Troubleshooting

**Tool reports unauthenticated.** Tell the user to run `/mcp` → xhost → Authenticate (Claude Code) or reconnect the connector (claude.ai). There is no token to set as an env var.

**"app name is already taken"** — they already own one with that name. Pick a different name or delete the old app with `mcp__xhost__delete_app`.

**"invalid app name"** — name violates the DNS-label rules or starts with a reserved prefix.

**Channel `status: provisioning` after deploy** — deploy is still running. Check `get_deploy_log`.

**`status: failed`** — read the deploy log; surface the failure to the user in plain language and propose a fix.

**Deploy fails right after start / log says `health check returned non-200`** (app template) — the app started but `/` didn't return a 200 on `$PORT` within 120s. Usual causes: the server bound `localhost` or a hardcoded port instead of `0.0.0.0:$PORT`; there's no `/` route (an API under `/api` only); the boot was too slow; or `install.sh`/the process ran out of memory. Fix the bind/`$PORT`, add a `/` handler returning 200, or slim the install.

**Local `git push` succeeded but the deploy is empty / nothing changed** — prod is bound to `branch:master`, but a fresh `git init` defaults to `main`. Push `master` (`git push xhost HEAD:master`) or deploy with `ref` set to your actual branch.
