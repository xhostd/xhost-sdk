# Recipes

These are complete deployments that you can follow from start to end. Each
recipe takes one shape of project: a static site, an API, or a background
worker. It then shows the full path. You create the app, write the files, set
the environment, deploy, and check that the app is live. A coding agent can
follow a recipe for you. The fastest method is to give your agent the recipe
and say "do this".

If your agent has no connection to xhost, start with
[Getting Started](https://docs.xhostd.com/getting-started). Then come back
here.

[Push code with git](https://docs.xhostd.com/guides/git) tells you about the
credential, the remote URL and the `git push` command.

## The recipes

The table lists the available guides. Each guide is complete in itself. You do
not need to read the other guides first.

| Guide | You get | Status |
|---|---|---|
| [Static site](https://docs.xhostd.com/guides/recipes-static) | An HTML/CSS/JS site on an HTTPS URL | Ready |
| [Node.js app](https://docs.xhostd.com/guides/recipes-app-node) | A JSON API on Express | Ready |
| [Python app](https://docs.xhostd.com/guides/recipes-app-python) | A JSON API on FastAPI | Ready |
| [Docker app](https://docs.xhostd.com/guides/recipes-docker) | Any runtime, from your own Dockerfile | Ready |
| [Postgres](https://docs.xhostd.com/guides/recipes-postgres) | A database, with migrations that run on deploy | Ready |
| [File uploads](https://docs.xhostd.com/guides/recipes-blob) | An app that stores files in object storage | Ready |
| [Sign in with Google](https://docs.xhostd.com/guides/recipes-oauth) | An app that knows who its visitors are | Ready |
| [Raw TCP](https://docs.xhostd.com/guides/recipes-port-forwarding) | A public `host:port` for a service that is not HTTP | Ready |
| [Background worker](https://docs.xhostd.com/guides/recipes-worker) | A long-running process, not a web server | Ready |
| [Ship without git](https://docs.xhostd.com/guides/recipes-commit-files) | A site put onto the app by tool call, where there is no shell to run `git` in | Ready |
| [Best practices](https://docs.xhostd.com/guides/bkm) | The habits that keep a deploy free of surprises | Ready |
| [Diagnose a slow app](https://docs.xhostd.com/guides/diagnose-slowness) | The cause of a slow or unhealthy channel, and the action for it | Ready |
| Custom domain | Your own domain, with HTTPS | Coming soon |

## The parts of a recipe

Every recipe has the same start point and the same end.

### What you need first

You need an xhost account, and an agent that can reach the xhost tools. You
need nothing more. There is no local runtime, no build step on your machine,
and no server to rent.

### What the transcripts show

Every tool call, every deploy log and every `curl` in a recipe is verbatim
output. It comes from a real run on a real account: the `docs` demo account,
whose apps stay up. Thus a hostname such as `recipe-static-docs.xhostd.com` is
a live address that you can visit. It is not a form for you to complete. If you
follow the recipe yourself, you get the same app under your own account name.

Where a recipe shortens a value, hides a value, or gives prose instead of a
capture, it says so at that place. The credentials are always placeholders. One
or two failure paths are in prose, because a real failure on a public demo
leaves the demo in a bad configuration.

The [Raw TCP](https://docs.xhostd.com/guides/recipes-port-forwarding) recipe is
different in one way. Its app is up on the demo account, the same as the other
apps. The run then released the public `host:port` in its transcript, on
purpose. That address therefore does not answer. It is the one captured
address in these guides that you must not use. Each new exposure gets a new
address.

### How it ends

Every recipe ends with a live address, and the one command that proves the
address works. For an HTTP recipe, that command is:

```bash
curl -sS https://<app>-<account>.xhostd.com/
```

If the command prints your app's response, the recipe worked. If it does not,
the recipe tells you which check to run next.

Two recipes end in another way, because each one deploys a service that uses no
HTTP. The [Raw TCP](https://docs.xhostd.com/guides/recipes-port-forwarding)
recipe proves itself with a socket round trip to a `host:port`. The
[Background worker](https://docs.xhostd.com/guides/recipes-worker) recipe has no
address that answers, so it proves itself with `get_runtime_log`, which counts
the work the worker did. The HTTPS hostname of each one returns 502 by design.

> A recipe shows the correct path, and also the two or three usual faults. A
> recipe is not a reference. [MCP Tools](https://docs.xhostd.com/mcp-tools) and
> the [API Reference](https://docs.xhostd.com/api) give the full lists of the
> tools and the endpoints.
