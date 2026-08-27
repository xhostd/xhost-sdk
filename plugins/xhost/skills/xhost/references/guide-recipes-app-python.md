# Recipe: Python API on the app template

## What you get

A Python 3.13 FastAPI service on an HTTPS URL. The build installs the
dependencies into the image. At boot, the platform starts uvicorn as a non-root
user. You write two shell scripts: `install.sh` runs one time at build time,
and `launch.sh` starts the server. The platform supplies the base image, the
Dockerfile, the port and the TLS.

The example app is live at
[recipe-python-docs.xhostd.app](https://recipe-python-docs.xhostd.app/). The
transcript in this guide deployed the four files below. The app has two
read-only routes, and it writes to no store. If you obey this recipe on your
own account, you get the same API under your own name.

## The files

Four files at the repo root.

### app.py

```python
from fastapi import FastAPI

app = FastAPI()


# The health check probes GET / and needs a 2xx. A pure API whose routes
# all live under /api fails the deploy even though the process is running,
# unless it creates the file named by $XHOST_READY_FILE instead.
@app.get("/")
def root():
    return {"ok": True, "service": "recipe-python-fastapi"}


@app.get("/api/echo")
def echo(q: str | None = None):
    return {"echo": q}
```

The file name and the variable name are important. `launch.sh` starts
`app:app`, which means "the object with the name `app` in the module `app`". If
you rename the file, you must also change that argument.

### requirements.txt

```
fastapi==0.115.6
uvicorn==0.34.0
```

Pin every dependency to an exact version. Without the pins, two deploys of the
same commit can install different code. The deploy that fails is then the
deploy that you did not change.

### install.sh

```sh
#!/bin/sh
# Runs at BUILD time, as root. --system installs into the image's own
# interpreter, so launch.sh needs no virtualenv activation.
set -eu

uv pip install --system --no-cache -r requirements.txt
```

### launch.sh

```sh
#!/bin/sh
# Runs at BOOT, as the non-root 'app' user. Never install anything here.
set -eu

exec uvicorn app:app --host 0.0.0.0 --port "$XHOST_HTTP_PORT"
```

Two flags hold the contract with the platform. `--host 0.0.0.0` makes the
server available from outside its container. The health check and the proxy
both come from outside the container. By default, uvicorn binds to an address
that they cannot reach.

`--port "$XHOST_HTTP_PORT"` reads the port that the platform assigned. Never
hardcode a port number. The `exec` makes uvicorn PID 1, so uvicorn gets the
stop signals directly. A shell in front of uvicorn ignores those signals.

## The deploy

Every app owns a git repo, and **`git push` → `deploy` is the standard path**.
A push sends only the diff. Thus each edit after the first one costs a few
lines, not the content of every file through a tool call. Use `commit_files` in
one situation only: git is not available on your machine. **A push stores your
code, but it does not deploy the code.** `deploy` is a separate call, so you
name the commit that goes live.

The deploy has four steps. If this is your first app, the
[static site recipe](https://docs.xhostd.com/guides/recipes-static) gives more
detail on the same four steps.

**1. Create the app.** This call creates the `prod` channel, its hostname and
its git repo.

```text
create_app(name="recipe-python", template="app")
→ {"id": "9e8f7bd3-a30d-45dc-9d01-7ef82c44e8a9",
   "name": "recipe-python",
   "template": "app",
   "repo_url": "https://git.xhostd.com/docs/recipe-python.git",
   "channels": [{"id": "44788022-ee4d-458d-802f-277cfda9116c",
                 "name": "prod",
                 "hostname": "recipe-python-docs.xhostd.app",
                 "current_sha": null}], ...}
```

Keep the app `id` and the `prod` channel `id`, because every later call takes
one or both.

**2. Mint a credential.** The token is your git password, and it is valid for
30 days.

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

Put the token in the **password** field of the remote URL:
`https://<username>:<token>@git.xhostd.com/<username>/<app>.git`.

**3. Clone, commit and push.** A new xhostd repo is empty, and git tells you so.
The `warning:` line below is correct, and it is not a fault:

```bash
$ git clone https://docs:$XHOST_TOKEN@git.xhostd.com/docs/recipe-python.git
Cloning into 'recipe-python'...
warning: You appear to have cloned an empty repository.
```

Write the four files into that directory. Then commit and push them:

```bash
$ cd recipe-python
$ git add -A
$ git commit -m "fastapi service on the app template"
$ git push origin master
```

Do not put the token in a file that you commit, and do not paste it.
`$XHOST_TOKEN` above is the value from `get_credentials`. A remote URL with a
real token goes into `.git/config`, into your shell history, and into the
output of every `git remote -v`.

**4. Deploy the branch.**

```text
deploy(app_id="9e8f7bd3-a30d-45dc-9d01-7ef82c44e8a9",
       channel_id="44788022-ee4d-458d-802f-277cfda9116c",
       ref="master")
→ {"deploy_id": "6c58aa48-faa9-422a-b4f2-3f668d247bf1",
   "channel_id": "44788022-ee4d-458d-802f-277cfda9116c",
   "status": "queued"}
```

`ref` is a branch name, and xhostd resolves it to the current head of that
branch. After a push, you do not need to know the sha. `deploy` also accepts
`sha` when you want an exact commit, and `sha` has priority if you give both.
The deploy then runs in the background. Use `get_deploy_log` to follow it.

## Verify it

```text
get_deploy_log(app_id="9e8f7bd3-a30d-45dc-9d01-7ef82c44e8a9",
               channel_id="44788022-ee4d-458d-802f-277cfda9116c",
               deploy_id="6c58aa48-faa9-422a-b4f2-3f668d247bf1")
```

This is that deploy. The log below shows the ids in a short form. It also omits
the buildkit lines and the ten dependency lines, because they teach nothing:

```text
[2026-07-31T17:33:49+00:00] deploy begin id=6c58aa48-... channel=44788022-... sha=65441d56...
[2026-07-31T17:33:49+00:00] git_sync ok: synced app=9e8f7bd3-... channel=44788022-... sha=65441d56...
[2026-07-31T17:33:49+00:00] [build] start sha=65441d56092e71c2e3e278b86b5901c39e2c8b7c
[2026-07-31T17:33:50+00:00] [build] #5 [1/3] FROM xhost-registry:5000/xhost-runtime:node22-py313@sha256:f63020a522e9...
[2026-07-31T17:33:50+00:00] [build] #5 CACHED
[2026-07-31T17:33:50+00:00] [build] #6 [2/3] COPY --chown=app:app . /app
[2026-07-31T17:33:50+00:00] [build] #7 [3/3] RUN if [ -f /app/install.sh ]; then chmod +x /app/install.sh && cd /app && ./install.sh && rm -f /app/install.sh; fi && chown -R app:app /app
[2026-07-31T17:33:50+00:00] [build] #7 0.156 Using Python 3.13.14 environment at: /usr/local
[2026-07-31T17:33:50+00:00] [build] #7 0.706 Resolved 12 packages in 548ms
[2026-07-31T17:33:50+00:00] [build] #7 0.741 Downloading pydantic-core (2.0MiB)
[2026-07-31T17:33:52+00:00] [build] #7 0.820 Prepared 12 packages in 113ms
[2026-07-31T17:33:52+00:00] [build] #7 0.844 Installed 12 packages in 24ms
[2026-07-31T17:33:52+00:00] [build] #7 0.844  + fastapi==0.115.6
[2026-07-31T17:33:52+00:00] [build] #7 0.844  + uvicorn==0.34.0
[2026-07-31T17:33:52+00:00] [build] #7 DONE 0.9s
[2026-07-31T17:33:53+00:00] [build] finished in 3s
[2026-07-31T17:33:53+00:00] [build] queue wait 0s, build 3s
[2026-07-31T17:33:53+00:00] [build] image 962.13 MB total, 13.48 MB charged — base xhost-runtime:node22-py313 exempt
[2026-07-31T17:33:54+00:00] channel snapshot saved: 0.00 MB
[2026-07-31T17:33:55+00:00] start_static_site template=app
[2026-07-31T17:33:56+00:00] health_check container=720d612faa30... port=3000 timeout=120.0s
[2026-07-31T17:33:56+00:00] [container] [xhost] starting launch.sh (XHOST_HTTP_PORT=3000) ...
[2026-07-31T17:33:58+00:00] [container] INFO:     Application startup complete.
[2026-07-31T17:33:58+00:00] [container] INFO:     Uvicorn running on http://0.0.0.0:3000 (Press CTRL+C to quit)
[2026-07-31T17:33:58+00:00] health_check ok
[2026-07-31T17:33:58+00:00] [container] INFO:     10.77.1.5:35710 - "GET / HTTP/1.1" 200 OK
[2026-07-31T17:33:58+00:00] caddy ensure_route hostname=recipe-python-docs.xhostd.app upstream=10.77.1.5:32042
[2026-07-31T17:33:59+00:00] deploy success
```

Read five facts in that log.

**`Installed 12 packages in 24ms`.** That speed is `uv`, and it is the reason
why `uv` is the standard here. This was the first build of the app. Thus `uv`
resolved and downloaded the two pins in `requirements.txt`: `Resolved 12
packages in 548ms`, with a 2 MiB `pydantic-core` wheel. The whole build, with
the image export, took three seconds.

**`Using Python 3.13.14 environment at: /usr/local`.** `--system` installed
into the image's own interpreter, not into a virtualenv. Thus `launch.sh` calls
`uvicorn` directly, and it activates no virtualenv first.

**`image 962.13 MB total, 13.48 MB charged`.** The platform counts only the
bytes that your build adds to the platform base. The base layers are exempt
from the image-size cap of your plan. Your twelve packages are the 13.48 MB.

**`channel snapshot saved: 0.00 MB`.** This is the automatic database snapshot
before the deploy. Every deploy that is not `static` makes one snapshot, also
when you use no database. Thus the size is zero here.

**`Uvicorn running on http://0.0.0.0:3000`, then `"GET / HTTP/1.1" 200 OK`.**
That 200 *is* the health check. The platform prints the `[xhost] starting
launch.sh (XHOST_HTTP_PORT=3000)` line above it, not your code. That line names
the port that the probe uses. `health_check ok` comes next. Only then does
`caddy ensure_route` point the hostname at the new container.

Then two commands prove the result against the live app:

```bash
$ curl -sS https://recipe-python-docs.xhostd.app/
{"ok":true,"service":"recipe-python-fastapi"}

$ curl -sS "https://recipe-python-docs.xhostd.app/api/echo?q=hello"
{"echo":"hello"}
```

Both responses have the type `application/json`. The first response is 45
bytes.

## When it goes wrong

### uvicorn binds 127.0.0.1

If you remove `--host 0.0.0.0`, uvicorn binds to `127.0.0.1`. Only a caller
inside the container can reach that address. The health check gets no answer,
it uses all of its 120 seconds, and the deploy fails on a timeout. At the same
time, the log shows a normal uvicorn start. The start-up line tells you which
address uvicorn used. Look for `Uvicorn running on http://0.0.0.0:...`, not for
`http://127.0.0.1:...`.

### There is no route at GET /

The health check asks for `/` on the health port, and it needs a 2xx or a 3xx.
An API with all its routes under `/api` answers 404 there, so the deploy fails
although the service works. The probe accepts one other signal: your app
creates the file with the name in `$XHOST_READY_FILE`. As an alternative, add a
route at `/`, however simple.

### You used pip in place of uv

`pip` works, but it is much slower on every build. `--system` is a `uv` flag,
and `pip install --system` is not valid. Thus a line that changes the tool but
keeps the flags fails immediately. Use `uv pip install --system --no-cache`.

### requirements.txt without pins

`fastapi` with no `==` resolves to the newest version at build time. Your next
deploy then installs a dependency upgrade that you did not request. The failure
then appears on a commit that changed no dependency.

### The ASGI path does not match

`uvicorn app:app` looks for a module `app` and an attribute `app` in that
module. If your file is `main.py`, the argument is `main:app`. If your FastAPI
object has the name `api`, the argument is `app:api`. A wrong argument makes
uvicorn exit immediately with an error about the ASGI app. The deploy then
fails on the health check.

[Recipe: Node API on the app template](https://docs.xhostd.com/guides/recipes-app-node)
is the Node equivalent of this recipe. If your app has no server-side code, the
[static site recipe](https://docs.xhostd.com/guides/recipes-static) is simpler.
