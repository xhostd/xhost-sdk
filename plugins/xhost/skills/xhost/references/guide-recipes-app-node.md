# Recipe: Node API on the app template

## What you get

A Node 22 Express service on an HTTPS URL. The platform builds it into its own
image at deploy time, and starts it as a non-root user at boot.

You write two shell scripts. `install.sh` runs once at build time. `launch.sh`
starts your server. The platform supplies the base image, the Dockerfile, the
port and the TLS.

The example is live at
[recipe-node-docs.xhostd.com](https://recipe-node-docs.xhostd.com/). It is the
four files below, and the transcript in this guide deployed them. The API has
two read-only routes, and there is no data store behind them. Do the recipe on
your own account, and you get the same API under your own name.

## The files

Four files at the repo root.

### package.json

```json
{
  "name": "recipe-node-express",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "dependencies": {
    "express": "4.21.2"
  }
}
```

Pin your dependencies to exact versions. A range means two deploys of the same
commit can install different code.

### server.js

```js
import express from "express";

const app = express();

// The health check probes GET / and needs a 2xx. An API whose routes all
// live under /api will fail the deploy even though the process is running,
// unless it creates the file named by $XHOST_READY_FILE instead.
app.get("/", (req, res) => {
  res.json({ ok: true, service: "recipe-node-express", uptime: process.uptime() });
});

app.get("/api/echo", (req, res) => {
  res.json({ echo: req.query.q ?? null });
});

const port = Number(process.env.XHOST_HTTP_PORT);
app.listen(port, "0.0.0.0", () => {
  console.log(`listening on 0.0.0.0:${port}`);
});
```

Two lines hold the full contract with the platform. `XHOST_HTTP_PORT` is the
port for your socket. Read that variable, and never write a number in the code.
`0.0.0.0` is the address for the bind. The health check and the proxy both reach
your container from outside it. **A server on `localhost` is therefore
unreachable, and the port makes no difference.**

### install.sh

```sh
#!/bin/sh
# Runs at BUILD time, as root. Output is baked into the image, so the
# install cost is paid once per deploy and never at boot.
set -eu

npm install --omit=dev --no-audit --no-fund
```

### launch.sh

```sh
#!/bin/sh
# Runs at BOOT, as the non-root 'app' user. Never install anything here.
# exec so the server is PID 1 and receives stop signals directly.
set -eu

exec node server.js
```

`install.sh` is optional, and `launch.sh` is necessary. This division is the
purpose of the `app` template. Each costly step occurs once, at build time, as
root. The boot then only runs `exec`.

## The deploy

Every app owns a git repo, and **`git push` → `deploy` is the standard path**.
A push sends only the diff. Each edit after the first therefore costs a few
lines, and not the full content of every file through a tool call. Use
`commit_files` in one situation only: the machine has no git. **A push stores
your code, but it does not deploy it.** `deploy` is a separate call, so the
commit that goes live is the commit that you name.

Four steps. The
[static site recipe](https://docs.xhostd.com/guides/recipes-static) gives more
detail on the same four steps, for your first app.

**1. Create the app.** This call makes the `prod` channel, its hostname and its
git repo.

```text
create_app(name="recipe-node", template="app")
→ {"id": "3ea579f7-1770-4e5f-84cb-595dad2a4fa6",
   "name": "recipe-node",
   "template": "app",
   "repo_url": "https://git.xhostd.com/docs/recipe-node.git",
   "channels": [{"id": "937e4d14-965c-4959-9adb-effc1555b5b7",
                 "name": "prod",
                 "hostname": "recipe-node-docs.xhostd.com",
                 "current_sha": null}], ...}
```

Keep the app `id` and the `prod` channel `id`, because each later call takes one
or both.

**2. Get a credential.** The token is your git password, and it is valid for
30 days.

```text
get_credentials()
→ {"token": "xh_...", "username": "docs",
   "expires_at": "2026-08-30T17:26:27Z",
   "scopes": ["blob:*", "deploy:*", "repo:*", "db:*", "channel:*"]}
```

**This recipe shows the HTTPS steps.** Where a shell is available, SSH is the
first git transport, because the private half of the key never enters a tool
call. One registration then covers every app on that machine. The
[git guide](https://docs.xhostd.com/guides/git) holds the transport branch and
the SSH commands.

Put it in the **password** field of the remote URL:
`https://<username>:<token>@git.xhostd.com/<username>/<app>.git`.

**3. Clone, commit and push.** A new xhost repo is empty, and git reports this.
The warning is correct, and it is not a fault:

```bash
$ git clone https://docs:$XHOST_TOKEN@git.xhostd.com/docs/recipe-node.git
Cloning into 'recipe-node'...
warning: You appear to have cloned an empty repository.
```

Write the four files into that directory. Then commit them and push them:

```bash
$ cd recipe-node
$ git add -A
$ git commit -m "express service on the app template"
$ git push origin master
```

Do not commit or paste the token. `$XHOST_TOKEN` above is the value from
`get_credentials`. A remote URL with a real token in it stays in `.git/config`,
in your shell history, and in the output of every `git remote -v`.

**4. Deploy the branch.**

```text
deploy(app_id="3ea579f7-1770-4e5f-84cb-595dad2a4fa6",
       channel_id="937e4d14-965c-4959-9adb-effc1555b5b7",
       ref="master")
→ {"deploy_id": "6bd8ff7c-1d29-4799-9410-b1f8e2f2c5db",
   "channel_id": "937e4d14-965c-4959-9adb-effc1555b5b7",
   "status": "queued"}
```

`ref` is a branch name. xhostd resolves it to the current head of that branch,
so after a push you do not need the sha. `sha` is also permitted, when you want
an exact commit, and `sha` has priority if you give both. The deploy then runs
in the background. Read its progress with `get_deploy_log`.

## Verify it

```text
get_deploy_log(app_id="3ea579f7-1770-4e5f-84cb-595dad2a4fa6",
               channel_id="937e4d14-965c-4959-9adb-effc1555b5b7",
               deploy_id="6bd8ff7c-1d29-4799-9410-b1f8e2f2c5db")
```

This is that deploy. The transcript shortens the ids, and it omits the buildkit
and npm-notice lines, because they teach nothing:

```text
[2026-07-31T17:30:45+00:00] deploy begin id=6bd8ff7c-... channel=937e4d14-... sha=1ba194d3...
[2026-07-31T17:30:45+00:00] git_sync ok: synced app=3ea579f7-... channel=937e4d14-... sha=1ba194d3...
[2026-07-31T17:30:45+00:00] [build] start sha=1ba194d3ea3be29fc7ab1a0c1a3ce2bbfbefb915
[2026-07-31T17:30:45+00:00] [build] queued 0s, starting
[2026-07-31T17:30:46+00:00] [build] #5 [1/3] FROM xhost-registry:5000/xhost-runtime:node22-py313@sha256:f63020a522e9...
[2026-07-31T17:30:46+00:00] [build] #5 CACHED
[2026-07-31T17:30:46+00:00] [build] #6 [2/3] COPY --chown=app:app . /app
[2026-07-31T17:30:46+00:00] [build] #7 [3/3] RUN if [ -f /app/install.sh ]; then chmod +x /app/install.sh && cd /app && ./install.sh && rm -f /app/install.sh; fi && chown -R app:app /app
[2026-07-31T17:30:47+00:00] [build] #7 1.731 added 69 packages in 2s
[2026-07-31T17:30:48+00:00] [build] #7 DONE 1.9s
[2026-07-31T17:30:50+00:00] [build] finished in 4s
[2026-07-31T17:30:50+00:00] [build] queue wait 0s, build 5s
[2026-07-31T17:30:50+00:00] [build] image 966.07 MB total, 17.41 MB charged — base xhost-runtime:node22-py313 exempt
[2026-07-31T17:30:51+00:00] channel snapshot saved: 0.00 MB
[2026-07-31T17:30:52+00:00] start_static_site template=app
[2026-07-31T17:30:53+00:00] health_check container=33ee729b559c... port=3000 timeout=120.0s
[2026-07-31T17:30:53+00:00] [container] [xhost] starting launch.sh (XHOST_HTTP_PORT=3000) ...
[2026-07-31T17:30:53+00:00] health_check ok
[2026-07-31T17:30:53+00:00] [container] listening on 0.0.0.0:3000
[2026-07-31T17:30:54+00:00] caddy ensure_route hostname=recipe-node-docs.xhostd.com upstream=10.77.1.5:32032
[2026-07-31T17:30:54+00:00] deploy success
```

Read these five lines with attention.

**`FROM xhost-runtime:node22-py313`.** The platform builds on its own base
image. That image has Node 22 and Python 3.13 together. You do not select the
image, and you do not write a Dockerfile.

**The `RUN` line is the Dockerfile that the platform writes.** The build copies
your repo to `/app`, with the owner `app:app`. Then `install.sh` runs as root,
so `apt-get` and a global `npm` work. The build then **deletes** `install.sh`,
so it can never run again at boot. The line `added 69 packages in 2s` is your
`npm install` in that step. This was the first build of the app, so the build
really installed the dependencies, and did not take them from a cache. The full
build still took five seconds.

**`image 966.07 MB total, 17.41 MB charged`.** Only the data that your build
adds to the platform base counts against your plan's image-size cap. The base
layers are exempt. A redeploy of the `app` template is therefore cheap.

**`channel snapshot saved: 0.00 MB`.** This is the automatic database snapshot
before the deploy. Every non-`static` deploy makes one, also when you do not use
the database. The size is therefore zero here.

**`[xhost] starting launch.sh (XHOST_HTTP_PORT=3000)`, then `health_check
ok`.** The platform prints that line, not your code. It gives the port that your
process must bind, immediately before `launch.sh` runs. Your own line
`listening on 0.0.0.0:3000` comes after it. Only after the probe passes does
`caddy ensure_route` point the hostname at the new container.

To find out if the app is alive, and to read no log, call `get_runtime_log` with
**no** `command`:

```text
get_runtime_log(app_id="3ea579f7-1770-4e5f-84cb-595dad2a4fa6", channel="prod")
```

A call with no command starts no container on the host, and it returns only the
status header. The header gives the process state, and the exit code if the
process stopped. It also gives an OOM-kill flag, the restart count, and the
container indices that you can read. Those indices are important after a
redeploy. The platform archives the log of the previous container, and does not
delete it. To read the output of the container that failed, give its
`container_index`, and a `command` such as `tail -n 200 app.log`.

Then the proof, in two commands, against the live demo:

```bash
$ curl -sS https://recipe-node-docs.xhostd.com/
{"ok":true,"service":"recipe-node-express","uptime":18729.900174679}

$ curl -sS "https://recipe-node-docs.xhostd.com/api/echo?q=hello"
{"echo":"hello"}
```

The two responses have the type `application/json`, and the first is 68 bytes.
The `uptime` value comes from `process.uptime()` in `server.js`. Your own
request to the demo therefore gives a different number from the transcript. The
container is the same one, and the difference is not a fault.

## When it goes wrong

### It listens on localhost instead of 0.0.0.0

`app.listen(port)` with no host, or with `"localhost"` or `"127.0.0.1"`, binds
only in the container. The health check then gets no answer. The probe waits its
full 120 seconds, and the deploy fails on the timeout. Your log still reports
that the server started. **Bind `"0.0.0.0"`.**

### There is no route at GET /

This is the most common failure on the first deploy of an `app` template. The
health check asks for `/` on the health port, and it needs a 2xx or a 3xx. An
API with all its routes under `/api` answers 404 there. The deploy then fails,
although the process operates correctly. The app can also create the file with
the name in `$XHOST_READY_FILE`, which is the other signal that the probe
accepts. If your app does not create that file, add a route at `/`, also a very
simple one.

### launch.sh installs things

`launch.sh` runs at each container start, as the non-root `app` user, on an
image that the platform built before. An `npm install` there is slow at each
boot. It can also fail on the permissions, and the platform discards its result.
Put each install in `install.sh`.

### launch.sh does not exec

Without `exec`, the shell stays PID 1, and your server is its child. The stop
signals then go to the shell, and the shell does not send them to your server.
Each redeploy therefore waits for the full kill timeout, and the server does not
stop correctly. Always use `exec` for the last command.

### The port is hardcoded

`app.listen(3000)` works, but it is still incorrect. The platform gives you the
port in `XHOST_HTTP_PORT`, and the platform selects the value. Read the
variable.

The Python equivalent of this recipe is
[Recipe: Python API on the app template](https://docs.xhostd.com/guides/recipes-app-python).
If your app has no server-side code, the
[static site recipe](https://docs.xhostd.com/guides/recipes-static) is simpler.
