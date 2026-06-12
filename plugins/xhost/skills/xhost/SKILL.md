---
name: xhost
description: >-
  Use when the user wants to deploy a website or app, host a static site, put
  something online, publish a page, create a preview URL, check deployment
  status, sign up for xhost, configure a token, or mentions xhost in any way.
  This skill handles everything: account setup, app creation, deploys, previews,
  and status checks.
tools: Bash, Read, Glob, Grep, Write
---

# xhost — Hosting Platform

xhost is a hosting platform for static sites and dynamic applications. You push code to a git remote, then trigger a deploy via the API. Every app gets a production HTTPS URL, and you can create preview URLs for branches.

## Current Context

- Working directory: !`pwd`
- Git remotes: !`git remote -v`
- Current branch: !`git branch --show-current`
- Latest commit: !`git log --oneline -1`

## Prerequisites

**Check for xhost MCP tools first.** This plugin bundles the xhost MCP server, so tools like `list_apps`, `create_app`, `commit_files`, and `deploy` (tool names contain `xhost`) are usually available directly. If they are:

- **Prefer the MCP tools for all API operations** — no token setup needed. Auth is handled by OAuth: if a tool call reports it's unauthenticated, tell the user to run `/mcp`, select the **xhost** server, and choose **Authenticate** (a browser opens for Google sign-in and a one-click approval; on first sign-in they pick a username).
- A token (`XHOST_TOKEN`) is only needed for the **fallback paths**: pushing to the git remote directly or calling the raw HTTP API with curl.

If MCP tools are NOT available, fall back to the raw HTTP API. Two environment variables control raw-API access:

- **XHOST_TOKEN** — Your API token (starts with `xh_`). Required for all raw-API operations and for the git remote.
- **XHOST_API_URL** — The API base URL. Defaults to `https://api.xhostd.com`.

Check if they are set:
```bash
[ -n "$XHOST_TOKEN" ] && echo "Token is set" || echo "Token not set"
echo "API URL: ${XHOST_API_URL:-https://api.xhostd.com}"
```

If the token is not set, guide the user through signup or login (see the Signup and Login sections below).

---

## Signup — Create a New Account

Account creation happens in a browser because Google sign-in is required.

- **MCP path (preferred):** tell the user to run `/mcp` → **xhost** → **Authenticate**. The browser flow covers signup too: Google sign-in, username selection on first sign-in, then a one-click approval. No token to copy.
- **Token path (for git pushes / raw curl):**
  1. Send the user to <https://xhostd.com> and tell them to click **Sign in with Google**.
  2. On first sign-in they pick a username (lowercase, digits, hyphens, 1–40 chars).
  3. Then send them to <https://xhostd.com/tokens?label=claude-code> and tell them to click **Create token** and copy the `xh_...` plaintext that's shown once.
  4. They give you the token; tell them:
     - `export XHOST_TOKEN=xh_...` (session)
     - Add to shell profile for persistence

If any raw API call later returns 401, the token is dead. Send the user to the same URL — <https://xhostd.com/tokens?label=claude-code> — to mint a new one, and have them paste it back. (For MCP tools, a 401 just means re-running `/mcp` → Authenticate.)

---

## Login — Configure an Existing Token

The user already has an xhost account and wants to use the raw API or git remote. (If they only need MCP tools, `/mcp` → Authenticate is enough — skip this.)

1. Ask the user for their `xh_*` token.
2. Validate by calling:
   ```
   curl -sf -H "Authorization: Bearer <token>" ${XHOST_API_URL:-https://api.xhostd.com}/apps
   ```
3. If invalid, tell the user. If valid, tell them to:
   - `export XHOST_TOKEN=xh_...` (session)
   - Add to shell profile for persistence

---

## Init — Create an App and Connect This Project

> **MCP shortcut:** if the xhost MCP tools are available, use `create_app` → `commit_files` → `deploy` instead of the raw-API steps below — no token or git remote required. The steps below are the raw-API/git fallback.

Set up a new xhost app linked to the current git repository.

1. **Check XHOST_TOKEN** — if not set, guide the user through signup or login first.
2. **Check XHOST_API_URL** — default to `https://api.xhostd.com`.
3. **Derive app name** from the current directory name. Slugify to DNS label rules (lowercase, replace non-alphanumeric with hyphens, trim, max 40 chars). Reserved prefixes: `git`, `api`, `www`, `admin`, `preview`, `staging`. Ask the user to confirm.
4. **Detect template and generate scripts** — follow this detection order:
   a. **launch.sh exists** — use `template: "app"`, do not overwrite. Tell user: "Found existing launch.sh."
   b. **Node.js project** — if `package.json` exists with `scripts.start`:
      - Generate `install.sh`:
        ```sh
        #!/bin/sh
        set -e
        if [ -f package-lock.json ]; then npm ci; else npm install; fi
        ```
      - Generate `launch.sh`:
        ```sh
        #!/bin/sh
        set -e
        if node -e "process.exit(require('./package.json').scripts?.build?0:1)" 2>/dev/null; then
            npm run build
        fi
        exec npm start
        ```
      - Tell user: "Detected Node.js project. Created install.sh and launch.sh."
      - Template: `"app"`.
   c. **Python project** — if `requirements.txt` exists:
      - Generate `install.sh`:
        ```sh
        #!/bin/sh
        set -e
        uv pip install --system --no-cache -r requirements.txt
        ```
      - Detect entry point: check for `app.py`, `main.py`, `manage.py` in order.
      - Generate `launch.sh`:
        ```sh
        #!/bin/sh
        set -e
        exec python <entry_point>
        ```
      - Template: `"app"`.
   d. **Otherwise** — template `"static"`. No scripts needed.
5. **Create the app**:
   ```
   curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/apps" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name":"<app_name>","template":"<template>"}'
   ```
6. Parse the response — it returns `id`, `repo_url`, `template`, and `channels` (with the prod channel's `hostname`).
7. **Add the git remote** — insert the token before the hostname in `repo_url`:
   ```
   git remote add xhost "https://${XHOST_TOKEN}@git.xhostd.com/<username>/<app>.git"
   ```
8. If no commits exist, stage everything and create an initial commit. If generated scripts aren't committed, stage and commit them.
9. **Push**:
   ```
   git push xhost master
   ```
10. **Deploy** — get the SHA and trigger the deploy:
    ```
    SHA=$(git rev-parse HEAD)
    curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels/<channel_id>/deploy" \
      -H "Authorization: Bearer $XHOST_TOKEN" \
      -H "Content-Type: application/json" \
      -d '{"sha":"'"$SHA"'"}'
    ```
11. Report: **"App created and deployed! Live at https://\<hostname\>"**
    - Mention that future deploys are: `git push xhost master` then use the Deploy flow below.
    - For app-template projects: first deploy takes 30-90s (install.sh runs).

---

## Deploy — Push and Ship

Commit, push, and trigger a deploy.

1. **Check XHOST_TOKEN** — if not set, guide through signup/login.
2. **Check xhost remote** — `git remote get-url xhost`. If missing, run the Init flow above.
3. **Check for uncommitted changes** — `git status --short`. If changes exist, stage all and commit with a descriptive message.
4. **Push**:
   ```
   git push xhost master
   ```
5. **Get the SHA**: `SHA=$(git rev-parse HEAD)`
6. **Find the app and channel** — list apps, match `repo_url` against the xhost remote URL (strip the token):
   ```
   curl -sf "${XHOST_API_URL:-https://api.xhostd.com}/apps" -H "Authorization: Bearer $XHOST_TOKEN"
   ```
7. **Trigger the deploy**:
   ```
   curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels/<channel_id>/deploy" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"sha":"<SHA>"}'
   ```
   Body accepts `{sha?, ref?}` (at least one required; both means `sha` wins).
   `{"ref":"master"}` works too — xhostd resolves the branch to its current HEAD.
8. **Check deploy log** (optional):
   ```
   curl -sf "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels/<channel_id>/logs?deploy=<deploy_id>" \
     -H "Authorization: Bearer $XHOST_TOKEN"
   ```
9. Report: **"Deployed! Live at https://\<hostname\>"**

---

## Preview — Deploy a Branch

Push the current branch and get a preview URL.

1. **Check XHOST_TOKEN** and **xhost remote** — same as Deploy.
2. **Get current branch**. If on `master`, warn: "This deploys to production. Consider creating a feature branch first." Ask if they want to continue.
3. **Find the app** via `GET /apps`, match against xhost remote.
4. **Check for an existing channel bound to this branch**:
   ```
   curl -sf "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels" \
     -H "Authorization: Bearer $XHOST_TOKEN"
   ```
5. If none exists, **create one for the branch**:
   ```
   curl -sf -X POST "${XHOST_API_URL:-https://api.xhostd.com}/apps/<app_id>/channels" \
     -H "Authorization: Bearer $XHOST_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name":"<branch>","git_ref_binding":"branch:<branch>"}'
   ```
6. **Commit, push the branch, then trigger deploy** with `{"ref":"<branch>"}` (xhostd resolves HEAD) or `{"sha":"<SHA>"}`.
7. The channel's hostname is `<branch>-<app>-<user>.<domain>`.
8. Report: **"Preview deployed! URL: https://\<preview_hostname\>"**

---

## Status — Check Apps and Deploys

1. **Check XHOST_TOKEN**.
2. **List apps**:
   ```
   curl -sf "${XHOST_API_URL:-https://api.xhostd.com}/apps" -H "Authorization: Bearer $XHOST_TOKEN"
   ```
3. If the `xhost` remote exists, show only the matching app. Otherwise show all.
4. For each app, display:
   - App name and template
   - For each channel: name, URL (`https://<hostname>`), branch binding, current SHA (first 7 chars), status

---

## Common Patterns

Use these across all operations:

- **API base URL**: `${XHOST_API_URL:-https://api.xhostd.com}`
- **Auth header**: `-H "Authorization: Bearer $XHOST_TOKEN"`
- **Error envelope**: `{"error": {"code": "...", "message": "..."}}` — always show `message` to the user
- **App resolution**: match the `xhost` git remote URL (strip token) against `repo_url` from `GET /apps`
- **Hostnames**: always present as `https://<hostname>`
- **Two-step deploy**: push stores code; `POST /apps/{id}/channels/{id}/deploy` triggers the build
- **DNS label rules**: `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`, max 40 chars
- **Reserved prefixes**: `git`, `api`, `www`, `admin`, `preview`, `staging`
- **Postgres for free**: every channel gets `DATABASE_URL` in its env automatically. Don't ask the user for connection strings; read `DATABASE_URL` from `process.env` (or equivalent) and let migrations run in `install.sh`. Reserved env keys (don't try to set): `DATABASE_URL`, `DATABASE_HOST`, `DATABASE_PASSWORD`.

## References

For complete API details, see the reference files in this skill:
- `references/api-reference.md` — Full endpoint documentation
- `references/getting-started.md` — Step-by-step new user guide
