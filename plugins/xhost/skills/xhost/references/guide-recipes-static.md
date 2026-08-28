# Recipe: static site

## What you get

You get a single HTML page on an HTTPS URL, and you run no process. The
`static` template gives the files that you commit to stock nginx, and mounts
them read-only. That is the whole app: no build step, no start command and no
process of your own to keep in service.

The example is live at
[recipe-static-docs.xhostd.app](https://recipe-static-docs.xhostd.app/). It is
the one file below, and the transcript in this guide deployed it. The page is
read-only, and there is nothing behind it to write to. If you do the recipe on
your own account, you get the same page under your own name.

## The files

One file, at the repo root.

### index.html

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Static landing — xhostd recipe</title>
<style>
  :root { color-scheme: light dark; }
  body {
    margin: 0; min-height: 100vh;
    display: grid; place-items: center;
    font: 16px/1.6 ui-sans-serif, system-ui, -apple-system, "Segoe UI", sans-serif;
    background: #0b0d12; color: #e6e9ef;
  }
  main { max-width: 34rem; padding: 2rem; }
  h1 { font-size: 1.75rem; margin: 0 0 .5rem; letter-spacing: -.02em; }
  p { margin: 0 0 1rem; color: #9aa4b8; }
  code {
    font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
    font-size: .9em; background: #171b24; padding: .15em .4em; border-radius: 4px;
  }
  a { color: #7aa2f7; }
</style>
</head>
<body>
<main>
  <h1>It's live.</h1>
  <p>
    This page is the whole app. The <code>static</code> template serves the
    files in your repo as-is through nginx — no build step, no start command,
    no process of your own.
  </p>
  <p>
    <code>index.html</code> at the repo root is what gets served at
    <code>/</code>. Add more files beside it and they are served at their
    matching paths.
  </p>
  <p><a href="https://docs.xhostd.com/guides">xhostd recipes</a></p>
</main>
</body>
</html>
```

nginx serves `index.html` at the repo root as `/`. It serves any other file
that you commit beside it at the same path.

## The deploy

Every app owns a git repo. **`git push`, then `deploy`, is the standard
path.** A push sends only the diff, which makes the second edit, the tenth
edit and the hundredth edit cheap. git transfers the few lines that changed.
It does not send the full contents of every file through a tool call. Use
`commit_files` in one situation only: git is not available on the machine that
you work on.

With both methods, you store the code in one step and you ship it in another
step. **A push stores your code; it does not deploy the code.** `deploy` is a
separate call, so you name the commit that goes live.

Four steps, in order.

**1. Create the app.** This call provisions the `prod` channel, its hostname
and its git repo.

```text
create_app(name="recipe-static", template="static")
→ {"id": "aea6786c-52ff-4ed3-bf07-ab3050a42069",
   "name": "recipe-static",
   "template": "static",
   "repo_url": "https://git.xhostd.com/docs/recipe-static.git",
   "channels": [{"id": "4e8973a5-2a78-4326-bd7b-f95506d84b9f",
                 "name": "prod",
                 "hostname": "recipe-static-docs.xhostd.app",
                 "current_sha": null}], ...}
```

Keep two ids: the app's `id`, and the `id` of the new `prod` channel. Every
later call takes one or both. A new app is not online: the hostname serves
nothing until a deploy completes. `repo_url` is the repo that you push to. If
you lose it, `get_app` returns it again.

**2. Mint a credential.** One token is your git password, your Postgres
password and your platform API bearer. It is valid for 30 days.

```text
get_credentials()
→ {"token": "xh_...", "username": "docs",
   "expires_at": "2026-08-30T17:26:27Z",
   "scopes": ["blob:*", "deploy:*", "repo:*", "db:*", "channel:*", "stats:read"]}
```

**This recipe shows the HTTPS steps.** Where a shell is available, SSH is the
first git transport, because the private half of the key never enters a tool
call. One registration then covers every app on that machine. The
[git guide](https://docs.xhostd.com/guides/git) holds the transport branch and
the SSH commands.

Put the token in the **password** field of the remote URL. This detail is
important:

```text
https://<username>:<token>@git.xhostd.com/<username>/<app>.git
```

**3. Clone, commit, push.** A new xhostd repo is empty, and git tells you so.
The warning is correct; it is not a fault:

```bash
$ git clone https://docs:$XHOST_TOKEN@git.xhostd.com/docs/recipe-static.git
Cloning into 'recipe-static'...
warning: You appear to have cloned an empty repository.
```

Write `index.html` into that directory. Then commit and push it:

```bash
$ cd recipe-static
$ git add -A
$ git commit -m "add landing page"
$ git push origin master
```

Do not put the token in a file that you commit, or in text that you paste.
`$XHOST_TOKEN` above holds the value from `get_credentials`. A remote URL with
a real token in it stays in `.git/config`, in your shell history and in the
output of every `git remote -v`.

**4. Deploy the branch.**

```text
deploy(app_id="aea6786c-52ff-4ed3-bf07-ab3050a42069",
       channel_id="4e8973a5-2a78-4326-bd7b-f95506d84b9f",
       ref="master")
→ {"deploy_id": "bbd8effc-a9f2-4b57-a5a1-dcdf830a3861",
   "channel_id": "4e8973a5-2a78-4326-bd7b-f95506d84b9f",
   "status": "queued"}
```

`ref` is a branch name. xhostd resolves it to the current head of that branch,
so after a push you do not need the sha. `deploy` also accepts `sha` when you
want an exact commit, and `sha` has priority if you pass both.

`deploy` returns as soon as it queues the deploy, and the work continues in
the background. Use `get_deploy_log` to follow it: the first line of its
reply states the outcome.

## Verify it

```text
get_deploy_log(app_id="aea6786c-52ff-4ed3-bf07-ab3050a42069",
               channel_id="4e8973a5-2a78-4326-bd7b-f95506d84b9f",
               deploy_id="bbd8effc-a9f2-4b57-a5a1-dcdf830a3861")
```

This is the reply for that deploy. A status header comes first, then the log.
The transcript has short ids, and it does not include the nginx entrypoint
lines:

```text
deploy bbd8effc — success (sha 206b94135aaa)
started: 2026-07-31T17:30:18Z   finished: 2026-07-31T17:30:19Z

[2026-07-31T17:30:18+00:00] deploy begin id=bbd8effc-... channel=4e8973a5-... sha=206b9413...
[2026-07-31T17:30:18+00:00] git_sync ok: synced app=aea6786c-... channel=4e8973a5-... sha=206b9413...
[2026-07-31T17:30:18+00:00] start_static_site template=static
[2026-07-31T17:30:19+00:00] start_static_site ok: container=c20e29eec53c...
[2026-07-31T17:30:19+00:00] health_check container=c20e29eec53c... port=80 timeout=10.0s
[2026-07-31T17:30:19+00:00] health_check ok
[2026-07-31T17:30:19+00:00] [container] 2026/07/31 17:30:19 [notice] 1#1: nginx/1.31.3
[2026-07-31T17:30:19+00:00] [container] 2026/07/31 17:30:19 [notice] 1#1: start worker processes
[2026-07-31T17:30:19+00:00] [container] 10.77.1.5 - - [31/Jul/2026:17:30:19 +0000] "GET / HTTP/1.1" 200 1297 "-" "Python-urllib/3.13" "-"
[2026-07-31T17:30:19+00:00] caddy ensure_route hostname=recipe-static-docs.xhostd.app upstream=10.77.1.5:32005
[2026-07-31T17:30:19+00:00] caddy ensure_route ok
[2026-07-31T17:30:19+00:00] pinned deployed sha 206b94135aaa92b47421e30a1b86efc6d6ed824f
[2026-07-31T17:30:19+00:00] deploy success
```

The first line states the outcome. `status` is one of `queued`, `running`,
`success`, or `failed`. Read the status from that header, not from the log
text. Poll `get_deploy_log` while the status is `queued` or `running` — the
log grows as the deploy progresses. `success` means the deploy is done, and
on `failed` the reason is in the log tail. `get_app` also shows an in-flight
deploy: the channel's `pending_deploy` field holds `{deploy_id, sha, status}`
until the deploy finishes, and `null` after.

Learn three points from that log.

**The log has no `[build]` line.** The deploy builds nothing, because this
template has nothing to build. The deploy syncs your commit and gives the
files to nginx. The whole log covers one second. An `app` deploy or a `docker`
deploy of the same commit writes about twelve build lines and an image-size
report. This template writes neither.

**`health_check ... port=80 timeout=10.0s`.** The platform probes a `static`
deploy on port 80, and gives it 10 seconds. nginx is already in service, and
you have no process to start. The platform probes the `app` template and the
`docker` template on port 3000, and gives them 120 seconds for your own
process to start. The probe itself is the line `"GET / HTTP/1.1" 200 1297`
below, from the nginx access log: **a static deploy is healthy only if `GET /`
answers 2xx or 3xx.** That line is below `health_check ok`, not above it. The
cause is the batch: the platform drains the container output into the log a
short time after the probe.

**`caddy ensure_route`.** The platform points the hostname at the new
container only after the health check passes. This log is a first deploy, so
there was no previous container to remove. On a second deploy, the line
`stop_and_remove old container=...` comes after the platform moves the route.
Thus the previous version serves the site until the new version passes its
health check. A failed deploy leaves your site in service.

The next commands prove it against the live demo:

```bash
$ curl -sS -o /dev/null -w '%{http_code}\n' https://recipe-static-docs.xhostd.app/
200

$ curl -sS -o /dev/null -w '%{http_code}\n' https://recipe-static-docs.xhostd.app/nope
404
```

The 200 carries `text/html` and 1297 bytes from `nginx/1.31.3`. That is the
same 1297 as the probe's access line, because it is the same file. The 404
comes from nginx. This template maps a URL path to a committed file and does
nothing else. Thus a path with no file behind it gives a 404; nginx does not
serve `index.html` instead.

## When it goes wrong

### No index.html at the repo root

The health check asks nginx for `/`. nginx answers 404, and the deploy fails.
nginx started correctly, but there is no file at the path of the probe. Commit
an `index.html` at the **repo root**. Do not put it in a `public/`, `dist/` or
`site/` subdirectory.

### A client-side route 404s

`https://recipe-static-docs.xhostd.app/nope` gives the nginx 404 above. A path
such as `/dashboard` gives the same 404 in a single-page app, although the
app's router expects to handle it. This template has no rewrite rule, and no
`try_files` fallback to `index.html`. nginx serves a path only if a committed
file is at that path, and answers 404 for all other paths. Write a real file
at each path that you link to. Or put the app on the `app` template, where
your own server decides what `/dashboard` means. Your server must then serve
`index.html` for those paths itself — the section "Single-page apps: the
fallback is yours" in
[the Docker recipe](https://docs.xhostd.com/guides/recipes-docker) shows the
fallback.

### You expected a build step

There is none. The `static` template copies nothing, runs nothing and compiles
nothing. It serves the bytes that you commit. If a bundler makes your site,
run the bundler on the machine where you write the code. Then commit the
**built output** to the repo root.

### You expected to run server-side code

The `static` template runs no process of yours. Thus it has no place for a
request handler, a database query or a background job. A page that needs one
of those belongs on the `app` template. See
[Recipe: Node API on the app template](https://docs.xhostd.com/guides/recipes-app-node)
or
[Recipe: Python API on the app template](https://docs.xhostd.com/guides/recipes-app-python).
