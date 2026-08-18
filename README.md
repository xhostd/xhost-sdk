# xhostd SDK

Claude Code plugin for [xhostd](https://xhostd.com) — deploy static sites and dynamic applications. Push code, get HTTPS URLs.

## Install

```
/plugin marketplace add xhostd/xhost-sdk
/plugin install xhost@xhost-sdk
```

Installing the plugin registers both the xhost skill and the remote MCP server (`https://mcp.xhostd.com/mcp/`).

After installing, reload plugins in your current session:

```
/reload-plugins
```

## Connect

Run `/mcp`, select **xhost**, and choose **Authenticate**. Your browser opens for Google sign-in — no token needed.

## Usage

Just use `/xhost` — it handles everything:

```
"deploy my website"          → signs up, creates app, pushes, deploys
"check my app status"        → shows apps, channels, URLs, deploy state
"create a preview for this branch" → pushes branch, creates preview URL
```

Or invoke it explicitly:

```
/xhost
```

The single `/xhost` skill handles account setup, app creation, deploys, previews, and status checks. Claude figures out what you need from context.

## What xhostd supports

- **Static sites** — nginx serves your HTML/CSS/JS
- **Node.js apps** — Express, Next.js, Fastify, Vite (give `install.sh` + `launch.sh`)
- **Python apps** — FastAPI, Flask, Django (give `install.sh` + `launch.sh`)
- **Docker apps** — any runtime, from a `Dockerfile` that you write
- **Background workers** — processes that run without end, not web servers

## Recipes

<https://docs.xhostd.com/guides> holds one complete recipe for each shape of app: every file, the exact calls, and the failure modes of that shape. Read the recipe for your shape before you write the code. The plugin carries the same recipes offline, under `plugins/xhost/skills/xhost/references/`.

## How it works

1. You push code to xhostd's git server
2. You trigger a deploy (explicitly, via `/xhost` or the API)
3. xhostd runs your `install.sh` (install the dependencies) then `launch.sh` (start the app on `$XHOST_HTTP_PORT`)
4. Your app is live over HTTPS with a wildcard cert

A Docker app runs the `CMD` in your `Dockerfile` in place of the two scripts. A static site needs no script, because nginx serves the committed files.

## Requirements

- Git installed locally
- An API token (from [console.xhostd.com/tokens](https://console.xhostd.com/tokens)) only if you push over git or call the API with raw curl — the MCP connection itself uses OAuth, no token

## License

MIT
