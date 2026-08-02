# Best known methods

Each recipe shows one shape of project from start to end. This guide carries
the advice that belongs to no recipe in particular. It tells you how to find
the cause of a fault. It names the stacks that give the least trouble. It shows
how to write code that stays correct after a redeploy. It also explains what
each platform error tells you to do.

This guide is for an agent that acts for a user. Every rule here comes with its
reason. A rule without a reason is easy to forget when it is not convenient.

## How to debug: the order to check

Do these checks in the given order. A different order costs you the most time.

**1. Did the deploy finish?** `get_deploy_log(app_id, channel_id, deploy_id)`
gives the record of the build, the health check and the route swap. If the
deploy failed, no other check helps. The old container still serves the
traffic, and the URL that you test gives you the previous version.

**2. Is the container alive now?** Call `get_runtime_log(app_id, channel)`
with **no** `command`. The platform starts no container to answer this call, so
it is the cheapest call available. It returns the status header only:

```
container #2 (xhost-fbae88c8-41dbea1f-00000002) — running
started: 2026-07-29T21:49:26.989002493Z
readable containers: #1, #2 (pass container_index to read an older one)
no command given — pass one (e.g. "tail -n 200 app.log") to read the log itself
```

That header tells you if the container stopped, and if it restarts again and
again. If the kernel stopped the container, the header says so. The exit-code
line then carries the note `(out of memory — the container hit its memory
limit)`. Read the header before you read one log line.

**3. Then read the log.** Pass a `command`. The platform runs the command on
the log itself:

```
get_runtime_log(app_id="...", channel="prod", command="tail -n 200 app.log")
```

The command runs in a temporary Debian container. The platform copies the
selected log to `/log/app.log`, and makes `/log` the work directory. The
container has no network, and no access to your app or its data. The platform
deletes the container after the command ends. These tools are available: `sh`,
`bash`, `grep`, `sed`, `awk`, `tail`, `head`, `cut`, `tr`, `sort`,
`uniq`, `wc`, `find`, `xargs`, `gzip`/`zcat`, `tac`, `nl`, `od`, `strings`,
`base64`, `perl`, `python3`, `node`.

There is no `jq`, no `rg` and no `less`. Write `grep -c ERROR app.log` instead.
Use `python3` if you must parse JSON. The platform stops the command after 30
seconds, and limits its output to approximately 256 KiB. Send the output
through `tail` or `head` to stay below that limit.

### Read the previous container's log

The line `readable containers: #1, #2` in that header is the useful part.
Container #1 is the version that you replaced. The platform archives the log of
a container before it removes that container. Thus, after a redeploy that made
the fault worse, you can still read why the old container was correct. You can
also read why the new container is not correct:

```
get_runtime_log(app_id="...", channel="prod",
                command="tail -n 200 app.log", container_index=1)
```

Without `container_index`, you get the highest available index. That index is
usually the live container.

### Make the log easy to read

The platform captures only the stdout and the stderr of your process. It merges
the two into one stream, in the order that your app wrote them. Each line stays
on its own line, and gets an RFC3339Nano timestamp at its start. An app that
writes its logs to a file in the container puts nothing here. Two more results
follow:

- **Write the level in the line yourself.** You can find `[error] upstream
  timeout` with `grep`. After the platform merges the two streams, a plain
  stack trace on stderr looks the same as every other line.
- **Do not buffer the output.** Python buffers stdout in blocks when stdout is
  not a terminal, and the platform does not set `PYTHONUNBUFFERED`. Thus a
  process that stops with a fault can lose its last and most important lines.
  Set the variable yourself:
  `set_env(app_id, key="PYTHONUNBUFFERED", value="1")`. Node does not have this
  problem. `PYTHONUNBUFFERED` is not a reserved key, so the platform accepts
  the write.

### When the health check fails

A failed health check gives a message that names every signal the probe
accepts. No recipe deploy failed a health check. The message below thus shows
the form that you see, without the container id. The real message carries the
full container id:

```
health check failed for container ...: no 2xx/3xx response at
GET / on port 3000 and no readiness file created at $XHOST_READY_FILE
within 120s
```

Read the message literally. A deploy passes on **one** of two signals, the
first signal that arrives. The first signal is an HTTP 2xx or 3xx response from
`GET /` on the health port. The second signal is the file that
`$XHOST_READY_FILE` names. The probe runs from outside the container, and it
does not follow a redirect. Thus a 301 response passes the check, but the probe
does not fetch the target of the redirect.

Work down this list:

- **The server binds the wrong address.** The probe comes from outside the
  container. The probe cannot reach a server on `127.0.0.1` or on `localhost`,
  whatever port that server took. Bind `0.0.0.0`.
- **The server binds the wrong port.** Read `XHOST_HTTP_PORT`. Never write the
  port into the code.
- **There is no route at `GET /`.** An API with all its routes below `/api`
  answers 404 at `/`. Add a simple route at `/`, or use the other signal. For
  the other signal, create the file `$XHOST_READY_FILE` when your app can
  serve.
- **The container stopped during the boot.** The platform checks the container
  state before both signals. Thus a process that creates the ready file and
  then stops still fails immediately. The message is `exited during boot (exit
  code N)`, and the platform does not wait for the timeout. After that message,
  read the runtime log. Do not use the health-check advice above.

The `static` template is the one exception. It gets no `XHOST_READY_FILE`, so
the HTTP probe is the only signal for a static site. Its timeout is 10 seconds,
not 120 seconds.

## Recommended stacks

The platform enforces nothing in this section. These combinations need the
fewest decisions, and they fail least often.

### On the app template

The base image carries Node 22 and CPython 3.13 together. It also carries `uv`,
`git` and a C toolchain. You do not write a Dockerfile, and you do not select
the base image.

**Python — FastAPI on uvicorn.** Put
`uv pip install --system --no-cache -r requirements.txt` in `install.sh`. The
`--system` flag puts the packages in the interpreter of the image. Thus
`launch.sh` has no virtualenv to activate. Start the app with
`exec uvicorn app:app --host 0.0.0.0 --port "$XHOST_HTTP_PORT"`.

**Node — Express or Hono.** Put
`npm install --omit=dev --no-audit --no-fund` in `install.sh`. Listen with
`app.listen(Number(process.env.XHOST_HTTP_PORT), "0.0.0.0")`.

For both stacks, pin every dependency to an exact version. With a range, two
deploys of the same commit can install different code. The deploy that then
fails has an empty diff.

`install.sh` runs one time at the build, as root. Put `apt-get`, global `npm`
and each compile step there. `launch.sh` runs at every container start, as the
non-root `app` user. Keep `launch.sh` to one `exec` line and nothing more.

### On the docker template

Select one of the warm base images. Your plan has an image cap. The platform
does not count a layer that your image shares with a platform base image. It
keeps these six images warm on every host:

| Warm base |
|---|
| `node:22-slim` |
| `node:24-slim` |
| `python:3.11-slim` |
| `python:3.12-slim` |
| `python:3.13-slim` |
| `debian:trixie-slim` |

Copy your dependency manifest and install the dependencies before you copy the
rest of the source. Write `COPY package.json package-lock.json ./`, then
`npm ci`, *then* `COPY . .`. A change to the source alone then reuses the
cached install layer, and does not install the dependencies again.

Do not put a secret in the build. The platform injects the environment
variables when the container runs, not when it builds the image. Thus a secret
in an `ARG` is not available at the build. If you do write a secret into the
image, it stays in a layer permanently.

### Both

Use `exec` for the final command. Without `exec`, the shell stays PID 1, and
your server is a child of the shell. The stop signals then go to the shell, and
the shell does not send them on. Thus every redeploy waits for a kill timeout,
and your app does not stop correctly.

## OAuth: know who the visitor is

The platform can sign a user in for you. It is a simple identity provider, and
it gives an identity token and nothing more. The edge **does not enforce the
identity**. A request with no identity still reaches your app. Your app must
decide what to do with that request.

After the sign-in, the browser carries a `__Host-xhost_id` cookie with an RS256
JWT in it. Verify that token correctly, every time:

- Get the public keys from `https://auth.xhostd.com/xhost-auth/jwks`, and
  select the key by the token's `kid`.
- Accept `alg` only if it is `RS256`. Never let the token's own header select
  the algorithm for you.
- Check that `iss` is `https://auth.xhostd.com`. Check that `aud` is your
  channel's hostname. Check that `exp` is in the future.
- The identity is in `sub`. The claims `email` and `name` are next to it.

If you decode the payload but do not verify the signature, you do not
authenticate the user. Any person can read a JWT. Any person can also forge a
JWT if you do not check the signature. Use a library that verifies the
signature. Cache the JWKS response; do not get it again for each request.

The full example is
[Recipe: Sign in with Google](https://docs.xhostd.com/guides/recipes-oauth).

## Put code onto the app

Every app owns a git repo. **`git push`, then `deploy`, is the standard path**
onto that repo. Use `commit_files` in one situation only: git is not available
on the machine that you work on.

The reason is the quantity of data that each path moves. A push sends only the
diff. Git and the server agree on what the remote already holds, and only the
difference goes over the network. A tool call cannot do this. A changeset
carries the full text of every file that it names, through the model's context,
every time.

The first commit of one file costs approximately the same on both paths. At the
tenth edit, the push is a few lines, but the tool call is the full project
again. Git also gives you what a changeset cannot: branches, a history that you
can read back, and a local copy that you edit in place.

**A push stores your code; it does not deploy the code.** These are two
separate operations, and the separation lets you name the commit that goes
live. After a push, call `deploy(app_id, channel_id, ref="master")`. The `ref`
value is a branch name, and xhostd finds the current head of that branch. Thus
you never need to know the sha. Pass `sha` instead when you want an exact
commit. If you pass both, `sha` wins.

**`sync_git` has no part in this path.** It refreshes an app's mirror of a
*connected GitHub repo*. It has no relation to a push to the app's own repo. If
an app has a connected GitHub source, the platform refuses `commit_files`, and
you put the code in GitHub.

One call gives you the credential. `get_credentials` returns a 30-day token.
Put the token in the **password** field of the remote URL. Never put the token
in a file that you commit.
[Recipe: static site](https://docs.xhostd.com/guides/recipes-static) shows the
full sequence, from the create step to the deploy step. The four steps are the
same on every template.

## Upgrade-safe code

The platform changes below your app. It builds the images again, it replaces
the containers, and it can move a channel's database. Code that expects no
change fails on a day when you changed nothing.

**Read the injected environment; never build it again yourself.** The platform
makes `DATABASE_URL` at every deploy from the current database endpoint of the
channel. The `S3_*` values give the gateway that your container must use. If
you build a connection string from values that you kept, it is correct only
until the platform moves something.

**Use `XHOST_HTTP_PORT`.** `PORT` is a deprecated alias with the same value.
The platform injects `PORT`, but it will remove the alias. New code must never
read `PORT`.

**Do not write to the container file system and expect the data later.** Every
deploy starts a new container from a new image. Your app can write data at the
runtime. If that data is not in the database or in the blob storage, the next
deploy loses it. Temporary files and caches are safe there. State is not safe
there.

**Do not write a container's identity into your code.** The container names,
the indexes and the internal IPs change at every deploy.

## Migration discipline

A database schema change has one correct place: the start of your launch
command. The change then runs when the container boots.

```sh
#!/bin/sh
set -eu

alembic upgrade head
exec uvicorn app:app --host 0.0.0.0 --port "$XHOST_HTTP_PORT"
```

On the `docker` template, put the same command in `CMD`:

```dockerfile
CMD ["sh", "-c", "alembic upgrade head && exec uvicorn app:app --host 0.0.0.0 --port $XHOST_HTTP_PORT"]
```

A migration cannot run at the build. The build has no environment variables,
and thus no `DATABASE_URL`. A `RUN alembic upgrade head` line fails
immediately.

Two rules follow from the deploy sequence.

**Write a migration that is safe for the previous version of your code.** The
new container boots, it migrates the schema, and it passes its health check.
The old container serves the traffic during that time, and the route changes
only after that. For some seconds, the old code runs against the new schema. A
new column, a new nullable column and a new index are all safe. If you drop or
rename a column that the old code selects, the old code gives errors in those
seconds. Split such a change into two deploys: first stop the use of the
column, then drop it.

**Make each migration idempotent, and let a failure stop the process.** On a
database that is already correct, `alembic upgrade head` does nothing. That is
the correct result, because the command runs at every container start, and also
at every restart. If the migration fails, let the process exit. If your code
hides a failed migration, you serve new code against an old schema, and that
result is worse than a failed deploy.

## Secrets and environment variables

Set a secret with `set_env(..., secret=True)`. The MCP tools can never read a
secret back, and `list_env` returns the metadata only. The tools can read a
plain variable back. The platform encrypts both at rest. The difference is who
can see the value later.

**Never commit a secret to the repo.** The build copies your repository into
the image. On the `app` template, the Dockerfile that the platform makes does
the copy. On the `docker` template, your own `COPY` line does it. A key that
you commit one time stays in a layer while that image exists. The next commit
cannot remove the key from that layer.

**The platform rejects a reserved key; it does not ignore the key without a
message.** The platform injects these keys, and `set_env` refuses to write
them: `XHOST_USER`, `XHOST_SHA`,
`XHOST_HTTP_PORT`, `PORT`, `XHOST_FORWARD_PORT`, `XHOST_READY_FILE`,
`DATABASE_URL`, `DATABASE_HOST`, `DATABASE_PASSWORD`, `S3_ENDPOINT`,
`S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_REGION`. If you
want a different database, use a variable with a different name.

**A change to the environment takes effect at the next deploy, not
immediately.** The platform fixed the environment of the live container when it
created that container. Set the value, then deploy.

**A channel override has priority over an app default.** `set_env` without
`channel` sets the default that every channel shares. `set_env` with
`channel="prod"` sets an override for that channel only. Use the app level for
the values that are the same everywhere. Use the channel level for the few
values that are different.

## Blob storage

If a channel has object storage, the platform injects `S3_ENDPOINT`,
`S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` and `S3_REGION`. Point
an S3-compatible client at these values.

**Use plain keys with no prefix.** Write `avatars/u123.png`, not
`myapp/prod/avatars/u123.png`. Each channel already has its own private prefix.
The gateway adds that prefix for you, and removes it for you. Your own
namespace makes the keys longer and gives no benefit.

**`.xhost/` is reserved.** The keys below it belong to the platform.

**The store always keeps the old versions.** If you write over a key, the store
keeps the old version. If you delete a key, the store writes a delete marker.
This behaviour makes a recovery from a bad write possible. It has two results.
A deleted object still exists until its old versions expire. Your usage figure
counts only the live bytes, that is, the newest version of each key that is not
deleted.

**There is no public object URL.** A browser cannot download an object
directly. Serve each object through your app. Your app then decides who can see
the object.

## Budgets: know your plan's limits

Each limit fails in a different way. Thus you must know which limit you are
near.

The platform counts the blob storage for the full account, across every
channel. It applies the other limits per container or per image.

| Plan | Channels | Memory per container | Visible cores | Blob storage | Charged image size |
|---|---|---|---|---|---|
| basic | 5 | 128 MB | 1 | 1 GiB | 512 MiB |
| builder | 10 | 512 MB | 2 | 5 GiB | 2 GiB |
| indie | 25 | 1024 MB | 4 | 50 GiB | 4 GiB |
| pro | 75 | 3072 MB | 8 | 150 GiB | 12 GiB |

**The memory column applies to the live container, not to the build.** A build
runs under a separate 4 GB budget. Thus an image that builds correctly can
still run out of memory when it boots on a small plan. The kernel then stops
the container. A 128 MB container gives much less memory than the build
that made the image. The status header of the runtime log tells you this. Its
exit-code line carries the note `(out of memory — the container hit its memory
limit)`.

**The charged image size is not the total image size.** The largest platform
base image that matches is exempt. The platform counts only what your build
adds on top. A deploy log line gives both numbers: `image 966.07 MB total,
17.41 MB charged — base xhost-runtime:node22-py313 exempt`. Read the second
number, not the first.

**The visible cores are not the CPU time.** The visible-cores number limits how
many cores the scheduler can use for your container. A separate quota limits
how much CPU time the container gets. On the basic plan, that quota is one
fifth of one core. Four visible cores do not give the work of four cores.

**Set the worker count yourself.** `nproc` reports the visible cores of your
plan correctly, but `/proc/cpuinfo` still shows the full topology of the host.
Thus a library that sizes a worker pool from `/proc/cpuinfo` starts far too
many workers and uses all the memory. Write `--workers 2`; do not let a library
select the number.

**Port forwarding is not on every plan.** The basic plan does not include it. A
request for it on the basic plan returns a plan error, not a port.

## Read a quota error

The platform makes its errors different on purpose, because each error needs a
different response. Do not retry all of them.

**402 `plan_limit_exceeded` — the plan does not permit this operation.** There
are too many channels, or the plan does not carry the feature. A retry never
works. Tell the user which limit the operation hit, and that an upgrade of the
plan is the correction.

**403 `permission_denied` — the operation is permitted, but not from this
interface.** Some controls are console-only on purpose. The console alone can
turn on external access to the database or to the object storage. The console
alone can manage the project members, and change the owner of a project. There
is no tool for these controls, and there must not be one. Tell the user to make
the change in the web console.

**409 `channel_busy` or a conflict — another operation is active.** A deploy or
a restore already holds the channel. You can retry this error. Wait for the
active operation to end, then try again.

**507 — the object storage is full.** The platform refuses a `PUT` that would
put the account above its blob limit. Delete the objects that you do not need,
then retry. The free space becomes available in approximately one minute. The
store keeps the old versions, so a delete of a key does not free its old
versions immediately.

## The undo path

Read this section before you need it.

**The platform takes a database snapshot before every deploy that is not
`static`.** It takes the snapshot whether you use the database or not. The
deploy log shows `channel snapshot saved`. You prepare nothing in advance.

**To roll the data back:** call `list_channel_snapshots(app_name, channel)` to
see the snapshots. Then call
`restore_channel_db(app_name, channel, snapshot_id)`.

The restore renames the live schema, and restores the snapshot into a new empty
schema. It drops the renamed copy only after the restore succeeds. Thus a
failure in the middle renames the live schema back, and loses nothing.

The platform refuses a restore of `prod` unless the app's environment contains
`XHOST_ALLOW_PROD_RESTORE=1`. That control is deliberate, and it is not a
formality. A restore also returns `channel_busy` if a deploy on that channel is
in the queue or is active.

**To roll the code back:** call `rewind(app_id, channel_id)`. It moves the
channel to the image of the last successful deploy with a commit that is
different from the live commit. It is fast, because it boots an image that the
platform keeps. There is no new build and no git sync. `rewind` is not
available on a channel that deployed one commit only. To go back more than one
step, `deploy` the older sha instead.

**The two operations are separate, and the difference is important.** `rewind`
changes the live image and nothing more. It does not reverse a migration. If
the bad deploy changed the schema, a rewind leaves the old code with a new
schema. Restore the database as well, or deploy a correction. `rewind` is not
available on the `static` template, because that template keeps no image.

**Only the console can restore the blob storage.** There is no MCP tool for it.
If you must recover the objects from an earlier point in time, use the web
console.

The [recipes index](https://docs.xhostd.com/guides) lists the recipes for all
of this. Start with the
[static site](https://docs.xhostd.com/guides/recipes-static). Then read the
[Node](https://docs.xhostd.com/guides/recipes-app-node) and
[Python](https://docs.xhostd.com/guides/recipes-app-python) app templates.
