# Recipes

Worked, end-to-end deployments you can follow start to finish. Each recipe
takes one shape of project — a static site, an API, a background worker — and
walks the whole path: create the app, write the files, set the env, deploy,
and check it is live. They are written for a coding agent to follow on your
behalf, so the fastest way to use one is to point your agent at it and say
"do this".

If you have not connected xhost to your agent yet, start with
[Getting Started](https://docs.xhostd.com/getting-started) and come back here.

## The recipes

Guides land here as they are written. Each one is self-contained: you do not
need to have read the others.

| Recipe | You get | Status |
|---|---|---|
| Static site | An HTML/CSS/JS site on an HTTPS URL | Coming soon |
| Node.js API | A JSON API with a Postgres database | Coming soon |
| Python API | A FastAPI service with a Postgres database | Coming soon |
| File uploads | An app that stores files in object storage | Coming soon |
| Sign in with Google | An app that knows who its visitors are | Coming soon |
| Custom domain | Your own domain, with HTTPS | Coming soon |
| Docker app | Any runtime, from your own Dockerfile | Coming soon |
| Background worker | A long-running process, not a web server | Coming soon |
| Best practices | The habits that keep a deploy boring | Coming soon |

## How a recipe is written

Every recipe assumes the same starting point and ends the same way.

### What you need first

An xhost account and an agent that can reach the xhost tools. Nothing else —
no local runtime, no build step on your machine, no server to rent.

### How it ends

Every recipe finishes with a live URL and the one command that proves it:

```bash
curl -sS https://<app>-<user>.xhostd.com/
```

If that prints your app's response, the recipe worked. If it does not, the
recipe tells you which check to run next.

> Recipes show the happy path plus the two or three ways it usually goes
> wrong. They are not a reference — the full tool and endpoint lists live in
> [MCP Tools](https://docs.xhostd.com/mcp-tools) and the
> [API Reference](https://docs.xhostd.com/api).
