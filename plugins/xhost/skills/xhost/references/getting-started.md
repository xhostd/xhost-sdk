# Getting Started with xhostd — Worked Example

This walks through what an agent does, end-to-end, when a non-technical user says something like *"build me a little site that lists my favourite coffee shops in Lisbon and put it online"*. Nothing for the user to install, paste or configure — the agent mints the credential and runs the commands.

## 0. Prerequisites

The xhostd plugin is installed (Claude Code) or the connector named `xhost` is enabled (claude.ai). The `mcp__xhost__*` tools are available. The user has authenticated once via OAuth — if not, point them to `/mcp` → xhost → Authenticate (Claude Code) or the connector's Connect button (claude.ai). They will sign in with Google and, on the first sign-in only, pick a **username** (lowercase letters, digits, hyphens; 1–40 chars; cannot start or end with a hyphen). That username becomes part of every public URL.

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
      "hostname": "lisbon-coffee-alice.xhostd.app",
      "git_ref_binding": "branch:master",
      "current_sha": null,
      "status": "provisioning"
    }
  ]
}
```

Remember `app_id` (`f1e2…`) and the prod channel's `id` (`c0a1…`).

## 3. Write the site

Write the files into a working directory, then push them to the app's own repo. **`git push` → `deploy` is the standard path**, for the first commit and every one after it: a push sends only the diff, so each later edit stays incremental instead of round-tripping whole file contents through a tool call.

The full step list is in **Pushing code with git** in SKILL.md. Pick the path from what the machine can do, before you push — never after a failure. With a shell, **SSH is the first transport**: the private half of the key never enters a tool call, and one registration covers every app on the machine. Reuse `~/.ssh/xhost_ed25519` if it exists; otherwise make a keypair in a subprocess and register the public half once:

```
ssh-keygen -t ed25519 -N "" -f ~/.ssh/xhost_ed25519   # only if the file is absent
# then: mcp__xhost__register_ssh_key(public_key=<content of ~/.ssh/xhost_ed25519.pub>, label="this machine")
git init && git add -A && git commit -m "initial site"
git remote add xhost-ssh "git@git.xhostd.com:alice/lisbon-coffee.git"
GIT_SSH_COMMAND="ssh -i ~/.ssh/xhost_ed25519" git push xhost-ssh HEAD:master
```

An SSH push needs no token — the `repo_url` from step 2 gives the username and the app name.

Always use exactly the path `~/.ssh/xhost_ed25519` — never the project directory, and never a per-project, per-app or per-tool suffix. It sits in `$HOME`, so every Claude Code session, IDE window and project on the machine reuses the one key instead of registering another.

**HTTPS fallback:** where outbound port 22 is blocked, or the SSH push fails, call `mcp__xhost__get_credentials` for a 30-day token — one `xh_` secret that is the git password, the Postgres password and the platform API bearer at once — put it in the **password** field of the remote URL (`repo_url` came back in step 2), and push:

```
git remote add xhost "https://alice:<token>@git.xhostd.com/alice/lisbon-coffee.git"
git push xhost HEAD:master
```

`HEAD:master` on either transport, because prod is bound to `branch:master` while a fresh `git init` defaults to `main`. Never write the token into a file the user might commit. **Pushing stores the code; it does not deploy** — that is step 4.

**Fallback, one case only:** when git is not available on the machine you are working on — a runtime with no shell, such as the claude.ai connector — use `mcp__xhost__commit_files(app_id, message, files, ref="master")` instead. `files` is a `{path: content-or-null}` map: a string upserts, `null` deletes, and a path you don't name is left alone, so send only what is changing. It returns `{"sha": "abc123…"}`, which is what you then deploy. On GitHub-connected apps it is refused — push to GitHub instead. Worked example: <https://docs.xhostd.com/guides/recipes-commit-files>.

For an `app`-template project the push is the same; the repo needs `install.sh` (optional) and `launch.sh` (required) at its root. The deploy only succeeds if the app signals readiness within 120s, and there are **two ways to do that** — whichever comes first. So the process must:

- **Either serve HTTP:** **bind `0.0.0.0` on `$XHOST_HTTP_PORT`** — read `$XHOST_HTTP_PORT` from the environment, don't hardcode. (`$PORT` is still injected at the same value, so existing apps keep working, but it is deprecated and will be removed — use `$XHOST_HTTP_PORT` in new code.) `python app.py` with Flask's default `app.run()` binds `localhost` on a fixed port and will fail the check; pass the host and `$XHOST_HTTP_PORT` explicitly — and **answer `/` with a 200**: an API whose routes are all under `/api` fails the check even though it runs; add a minimal `/` handler.
- **Or create `$XHOST_READY_FILE`** — for a process with no HTTP surface at all (a queue consumer, a cron-style daemon), create the file at the injected path instead of adding a dummy listener; it can be empty. Do it once the work loop is genuinely running, not at the top of `launch.sh`. Such a project keeps its hostname, and that URL returns 502 — expected, not a failure.
- **Boot within 120s and stay within a small memory budget (~128 MB)** at run time — the cap applies to the running server, not to the build.
- **Run as a non-root user.** The container runs as `app`; writable paths are `/app`, `$HOME`, and `/tmp`. All installation must live in `install.sh`, which runs at **build** time as root — installing from `launch.sh` fails with `Permission denied`.

Example minimal pair (`install.sh` bakes deps into the image at build time, then `launch.sh` starts the server at boot):

```sh
# install.sh
#!/bin/sh
set -e
# Prefer uv over pip — same packages, dramatically faster.
uv pip install --system --no-cache flask gunicorn
```
```sh
# launch.sh
#!/bin/sh
set -e
exec gunicorn --bind "0.0.0.0:$XHOST_HTTP_PORT" app:app
```

## 4. Deploy

```
mcp__xhost__deploy(
  app_id="f1e2…",
  channel_id="c0a1…",
  ref="master",
)
```

Returns `{"deploy_id": "d…", "channel_id": "c0a1…", "status": "queued"}`. Deploys run async. `ref` is a branch name and xhostd resolves it to that branch's current head, so after a push you never need to know the sha; pass `sha="abc123…"` instead to pin an exact commit — which is what you do with the sha `commit_files` returned.

## 5. Watch the build

```
mcp__xhost__get_deploy_log(app_id="f1e2…", channel_id="c0a1…", deploy_id="d…")
```

Returns plain text. Poll until the log shows the build finished. `static` deploys take a few seconds; the first `app`-template deploy takes 30–90 seconds because `install.sh` runs.

## 6. Hand the user the URL

Read `hostname` from the prod channel (it was in step 2's response, and `mcp__xhost__list_channels` or `mcp__xhost__get_app` will return it later). Tell the user:

> Live at https://lisbon-coffee-alice.xhostd.app

## 7. Iterate

For each follow-up change — "add a section for tea shops", "fix the broken link" — edit the files in the checkout, `git commit`, `git push xhost HEAD:master`, then `deploy` with `ref="master"`. The same `prod` channel id is reused, and the remote is already configured, so a follow-up is a couple of shell commands and one tool call. This is where the standard path pays: the push carries only the lines that changed, so the tenth edit costs about what the first one did — where a `commit_files` changeset would carry the full text of every file it names, again, every time. On the fallback path (no shell), that is still the loop: `commit_files` with only the changed files, then `deploy` with the sha it returns.

To preview a change without touching production:

```
mcp__xhost__create_channel(app_id="f1e2…", name="draft", git_ref_binding="branch:draft")
git push xhost HEAD:draft
mcp__xhost__deploy(app_id="f1e2…", channel_id="<draft channel id>", ref="draft")
```

The preview is live at `https://draft-lisbon-coffee-alice.xhostd.app`.

## 8. Optional extras

- **Env vars & secrets:** `mcp__xhost__set_env(app_id, key="STRIPE_KEY", value="sk_…", secret=True)` — mark credentials `secret=True` (values never readable back through MCP; the reveal is a protected action that answers `protected_action` to an agent credential, and it is audit-logged), add `channel="staging"` for a per-channel override that wins over the app default at deploy time. `mcp__xhost__list_env(app_id, channel=…)` shows the resolved view. Every channel automatically has `DATABASE_URL` for its own Postgres database; don't set it.
- **Custom domain:** `mcp__xhost__add_custom_domain(app_name, channel, domain)` returns an `instructions` field with the exact TXT + CNAME/A records the user needs to add at their registrar — relay it verbatim. After they add the records, call `mcp__xhost__verify_custom_domain` with the same args. HTTPS works automatically once verified.
- **Google sign-in for parts of the site:** zero-config — no tool to call. `/xhost-auth/*` works on every channel. After sign-in the gateway sets a signed identity cookie `__Host-xhost_id` (an RS256 JWT) on the channel host; the app verifies it against the JWKS at `https://auth.xhostd.com/xhost-auth/jwks` (pin `RS256`, check `iss`/`aud`/`exp`) and gates its own routes. **xhostd does identity only, never edge gatekeeping — nothing is blocked at the edge, so every route stays public (anonymous visitors get `200`) until your app verifies the cookie and enforces access itself in code.** Send signed-out users to `/xhost-auth/login?return_to=<path>`. `__Host-xhost_id` is a reserved cookie name. Per-stack verify snippets: <https://docs.xhostd.com/oauth>.

## Troubleshooting

**Tool reports unauthenticated.** Tell the user to run `/mcp` → xhost → Authenticate (Claude Code) or reconnect the connector (claude.ai). There is no token to set as an env var.

**"app name is already taken"** — they already own one with that name. Pick a different name or delete the old app with `mcp__xhost__delete_app`.

**"invalid app name"** — name violates the DNS-label rules or starts with a reserved prefix.

**Channel `status: provisioning` after deploy** — deploy is still running. Check `get_deploy_log`.

**`status: failed`** — read the deploy log; surface the failure to the user in plain language and propose a fix.

**Deploy fails right after start / log says `health check failed for container …`** (app template) — neither readiness signal arrived within 120s: `/` didn't return a 2xx on `$XHOST_HTTP_PORT` *and* no file was created at `$XHOST_READY_FILE`. Usual causes: the server bound `localhost` or a hardcoded port instead of `0.0.0.0:$XHOST_HTTP_PORT`; there's no `/` route (an API under `/api` only); the boot was too slow; or `launch.sh` hit `Permission denied` — it runs as the non-root `app` user, so installing anything there, or writing outside `/app`/`$HOME`/`/tmp`, crashes it. Fix the bind/`$XHOST_HTTP_PORT`, add a `/` handler returning 200, or move the install into `install.sh`. If the project has no HTTP surface at all, create `$XHOST_READY_FILE` once its work loop is running instead of adding a listener.

**`git push` succeeded but the deploy is empty / nothing changed** — prod is bound to `branch:master`, but a fresh `git init` defaults to `main`. Push `master` (`git push xhost HEAD:master`) or deploy with `ref` set to your actual branch.
