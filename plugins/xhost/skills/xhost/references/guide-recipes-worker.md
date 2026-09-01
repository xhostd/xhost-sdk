# Recipe: a background worker

## What you get

A Python worker on the `app` template. It runs one loop without end, and it
serves no HTTP.

The deploy's health check accepts two readiness signals, and it takes the first
signal that arrives. The first signal is an HTTP 2xx or 3xx response from
`GET /` on `$XHOST_HTTP_PORT`. The second signal is the file with the name in
`$XHOST_READY_FILE`. A worker binds no port, so only the second signal is
available to it. Most recipes teach the first signal. This one teaches the
second, and it shows how to verify a channel that has no URL.

The app is `recipe-worker`, the template is `app`, and the channel is `prod`.
The channel keeps its HTTPS hostname, `recipe-worker-docs.xhostd.app`, and that
hostname returns 502. That is correct, and
[The HTTPS hostname returns 502](#the-https-hostname-returns-502) explains why.

**The worker's only output is its log.** An HTTP recipe proves itself with a
request to a URL. This one has no URL to request. `get_runtime_log` replaces
`curl` here, and every proof below comes from it.

**The container's local disk does not survive a redeploy.** Each deploy starts a
new container from a new image, and the old container goes away with its files.
Temporary files are safe there, and state is not. Keep durable state in a
database or in object storage:
[Postgres](https://docs.xhostd.com/guides/recipes-postgres) and
[File uploads](https://docs.xhostd.com/guides/recipes-blob) give you both.

## The files

Two files at the repo root. The worker uses only the standard library, so it
needs no `requirements.txt`. `install.sh` is optional, and this recipe has
none.

### worker.py

```python
"""A worker channel that runs a loop and serves no HTTP.

The work is a toy: a counter that prints one JSON line per tick. The shape
around it is the part to copy. This channel serves no HTTP at all, so the
deploy's HTTP probe can never pass. The worker creates $XHOST_READY_FILE
after it reads a valid configuration, which is the other signal the health
check accepts.
"""

import json
import os
import time
from datetime import UTC, datetime
from pathlib import Path


def emit(record):
    # The platform reads the container's stdout. Python buffers stdout when
    # it is not a terminal, so a worker that does not flush looks silent in
    # the runtime log. Flush every line.
    line = {"ts": datetime.now(UTC).isoformat(), **record}
    print(json.dumps(line), flush=True)


def setup():
    # A bad TICK_SECONDS must stop the worker here. main() creates the
    # ready file only after this function returns.
    interval = float(os.environ.get("TICK_SECONDS", "5"))
    if interval <= 0:
        raise ValueError(f"TICK_SECONDS must be positive, not {interval}")
    return interval, {"ticks": 0, "started": time.monotonic()}


def do_tick(state):
    state["ticks"] += 1
    uptime = time.monotonic() - state["started"]
    return {
        "event": "tick",
        "count": state["ticks"],
        "uptime_s": round(uptime, 1),
    }


def main():
    interval, state = setup()

    # Create the ready file HERE, not before. setup() can fail, and that
    # failure must fail the deploy. A worker that creates the file first
    # reports a healthy channel, then dies and restarts without end.
    Path(os.environ["XHOST_READY_FILE"]).touch()
    emit({"event": "ready", "interval_s": interval})

    # The loop never stops. If main() returns, the container exits, and the
    # platform restarts it. During the health check, that exit fails the
    # deploy.
    while True:
        try:
            record = do_tick(state)
        except Exception as exc:
            # One bad tick must not stop the worker. Report the error, then
            # go on to the next tick.
            reason = f"{type(exc).__name__}: {exc}"
            record = {"event": "error", "error": reason}
        emit(record)
        time.sleep(interval)


if __name__ == "__main__":
    main()
```

Four parts of that file are important.

**The worker creates the ready file after `setup()` returns, and never before.**
This is the key rule of the recipe. `setup()` reads `TICK_SECONDS`, and it
refuses a value that is not positive. A fatal configuration error therefore
stops the worker before the file exists, and the deploy fails.
[A fatal setup error stops the deploy](#a-fatal-setup-error-stops-the-deploy)
shows this on a real deploy. Create the file at the first correct moment: after
the configuration is valid, and the worker is able to work.

**`emit` flushes every line.** Python buffers stdout in blocks when stdout is
not a terminal, and the platform sets no `PYTHONUNBUFFERED`. A worker without a
flush therefore looks silent for minutes at a time. The `flush=True` argument
costs one word. [Read the log](#read-the-log) shows the result: those lines came
out of a worker that still ran.

**The loop never returns.** The container exits when `main` returns, even with
the exit code 0. The platform then starts the container again, because the
restart policy is `unless-stopped`. Inside the health window, that exit fails
the deploy. After the health window, the container restarts without end, and
the deploy stays successful.

**One bad tick does not stop the worker.** `do_tick` runs inside a `try`, and
the handler writes an `error` record for the tick that failed. The loop then
goes on. Without that handler, one exception in one tick ends the whole
process. Each record is one JSON line, so `grep` and `wc` are sufficient to
read the output of a long run.

### launch.sh

```sh
#!/bin/sh
# Runs at BOOT, as the non-root 'app' user. Never install anything here.
# There is no install.sh: this worker uses only the standard library.
set -eu

exec python3 worker.py
```

`exec` makes the Python process PID 1. The process then gets the stop signals
directly, and not through a shell that discards them. Without `exec`, the shell
stays PID 1, and every stop takes the full stop timeout.

## The deploy

Three steps.

### Step 1 — create the app

```text
create_app(name="recipe-worker", template="app")
→ {"id": "26a55c60-c6d0-421d-8d0f-f80e7db90f56",
   "name": "recipe-worker",
   "repo_url": "https://git.xhostd.com/docs/recipe-worker.git",
   "template": "app",
   "port_forwarding_enabled": false,
   "port_forwarding_available": true,
   "channels": [{"id": "59489f61-ad60-4275-88cd-a6a5a7688f9a",
                 "name": "prod",
                 "hostname": "recipe-worker-docs.xhostd.app",
                 "git_ref_binding": "branch:master",
                 "current_sha": null,
                 "status": "provisioning",
                 "pending_deploy": null}],
   "owner_username": "docs",
   ...}
```

Keep the app `id` and the `prod` channel `id`, because the deploy call takes
both. `status: "provisioning"` is the normal state of a new channel, and the
hostname is already allocated. This recipe needs no public TCP port, so the two
`port_forwarding_*` fields are not important here. The
[Raw TCP recipe](https://docs.xhostd.com/guides/recipes-port-forwarding)
explains them.

### Step 2 — push the files

Every app owns a git repo, and **`git push` → `deploy` is the standard path**.
Use `commit_files` only when the machine has no git. A deploy reads a commit
from the repo, and it does not know the source of that commit.

**This recipe shows the HTTPS steps.** Where a shell is available, SSH is the
first git transport, because the private half of the key never enters a tool
call. One registration then covers every app on that machine. The
[git guide](https://docs.xhostd.com/guides/git) holds the transport branch and
the SSH commands.

Your credential is the token from `get_credentials()`. Put it in the
**password** field of the remote URL:

```text
https://<username>:<token>@git.xhostd.com/<username>/<app>.git
```

A new xhostd repo is empty, and git reports this. The warning is correct, and it
is not a fault:

```bash
$ git clone https://docs:$XHOST_TOKEN@git.xhostd.com/docs/recipe-worker.git
Cloning into 'recipe-worker'...
warning: You appear to have cloned an empty repository.
```

Write `worker.py` and `launch.sh` into that directory. Then commit the two
files and push them:

```bash
$ cd recipe-worker
$ git add -A
$ git commit -m "tick worker that signals readiness with a file"
[master (root-commit) 706dbe6] tick worker that signals readiness with a file
 2 files changed, 75 insertions(+)
 create mode 100755 launch.sh
 create mode 100644 worker.py
$ git push origin master
To https://git.xhostd.com/docs/recipe-worker.git
 * [new branch]      master -> master
```

Do not commit or paste the token. `$XHOST_TOKEN` above is the value from
`get_credentials`. A remote URL with a real token in it stays in `.git/config`,
in your shell history, and in the output of every `git remote -v`.

### Step 3 — start the deploy

```text
deploy(app_name="recipe-worker",
       channel="prod",
       ref="master")
→ {"deploy_id": "1d923d57-9779-4ffa-a214-542a7d0217b1",
   "channel_id": "59489f61-ad60-4275-88cd-a6a5a7688f9a",
   "status": "queued"}
```

`ref` is a branch name. xhostd resolves it to the current head of that branch,
so after a push you do not need the sha. A push stores your code, and it starts
no deploy. `deploy` is always its own call.

## Verify it

### The deploy log

`get_deploy_log(app_name, channel, deploy_id)` returns the record: a status
header first, then the log. The first line states the outcome — poll while
the status is `queued` or `running`. Below is the end of deploy
`1d923d57-9779-4ffa-a214-542a7d0217b1`. The platform no longer holds this
deploy's row, so the header's `started` value comes from the log's own
timestamps. The `...` mark stands for the `[build]` lines above it:

```text
deploy 1d923d57 — success (sha 706dbe67795f)
started: 2026-08-02T15:24:39Z   finished: 2026-08-02T15:24:47Z

...
[2026-08-02T15:24:43+00:00] [build] image 948.68 MB total, 0.02 MB charged — base xhost-runtime:node22-py313 exempt
[2026-08-02T15:24:45+00:00] channel snapshot saved: 0.01 MB
[2026-08-02T15:24:45+00:00] start_static_site template=app
[2026-08-02T15:24:46+00:00] start_static_site ok: container=5013ff44274ae4b2e1bca63419915c4d3b8b6923d81a3f86002f2f2fba8065af
[2026-08-02T15:24:46+00:00] health_check container=5013ff44274ae4b2e1bca63419915c4d3b8b6923d81a3f86002f2f2fba8065af port=3000 timeout=120.0s
[2026-08-02T15:24:46+00:00] [container] [xhost] starting launch.sh (XHOST_HTTP_PORT=3000) ...
[2026-08-02T15:24:46+00:00] health_check ok
[2026-08-02T15:24:46+00:00] [container] {"ts": "2026-08-02T15:24:46.443804+00:00", "event": "ready", "interval_s": 5.0}
[2026-08-02T15:24:46+00:00] [container] {"ts": "2026-08-02T15:24:46.443942+00:00", "event": "tick", "count": 1, "uptime_s": 0.0}
[2026-08-02T15:24:47+00:00] caddy ensure_route hostname=recipe-worker-docs.xhostd.app upstream=10.77.1.5:32053
[2026-08-02T15:24:47+00:00] pinned deployed sha 706dbe67795ff3488823ef80be7be2a0855dc28e
[2026-08-02T15:24:47+00:00] deploy success
```

Read five facts in that log.

**`image 948.68 MB total, 0.02 MB charged`.** The build added almost nothing to
the warm base image. A worker that uses only the standard library therefore has
an image cost near zero.

**`start_static_site template=app`.** `start_static_site` is the platform's
internal name for this step, on every template. It is not a claim that this app
is a static site. The same line names the real template: `template=app`.

**`health_check ... port=3000 timeout=120.0s`, and `health_check ok` in the
same second.** The platform publishes the HTTP health port on every template,
and it probes that port. No process in this container binds port 3000, so the
HTTP arm can never answer. With the HTTP arm alone, the 120-second window
expires. The ready file gave the signal instead. `health_check ok` and the
worker's `ready` line share the second `15:24:46`.

**The `ready` line and the first `tick` line.** These two lines are the worker's
own output, and the `[container]` prefix marks them. `interval_s: 5.0` is the
default value from `setup`, because the channel set no `TICK_SECONDS`.

**`caddy ensure_route hostname=recipe-worker-docs.xhostd.app`.** The channel
gets its HTTPS route, the same as every other channel. Nothing listens behind
that route, so the hostname answers 502.

### The status header

`get_runtime_log` with no `command` reports what you can read, and it reads
nothing. It is the cheapest call available, because the platform starts no
container to answer it:

```text
get_runtime_log(app_name="recipe-worker", channel="prod")
→ container #1 (xhost-26a55c60-59489f61-00000001) — running
  started: 2026-08-02T15:24:46.182767751Z
  readable containers: #1 (pass container_index to read an older one)
  no command given — pass one (e.g. "tail -n 200 app.log") to read the log itself
```

Read this header before you read one log line. It names the container, its
state and its start time, and `started:` agrees with the deploy log above. Take
the list after `readable containers:` from your own output, and do not compute
it. The platform archives the log of a container before it removes that
container, so an older container stays readable.

### Read the log

Give the call a `command`, and it returns the log:

```text
get_runtime_log(..., channel="prod", command="tail -n 5 app.log")
→ container #1 (xhost-26a55c60-59489f61-00000001) — running
  started: 2026-08-02T15:24:46.182767751Z
  readable containers: #1 (pass container_index to read an older one)
  log file: /log/app.log (30616 bytes)

  2026-08-02T15:44:51.538721598Z {"ts": "2026-08-02T15:44:51.538414+00:00", "event": "tick", "count": 242, "uptime_s": 1205.1}
  2026-08-02T15:44:56.539203647Z {"ts": "2026-08-02T15:44:56.538822+00:00", "event": "tick", "count": 243, "uptime_s": 1210.1}
  2026-08-02T15:45:01.539740081Z {"ts": "2026-08-02T15:45:01.539214+00:00", "event": "tick", "count": 244, "uptime_s": 1215.1}
  2026-08-02T15:45:06.540191946Z {"ts": "2026-08-02T15:45:06.539805+00:00", "event": "tick", "count": 245, "uptime_s": 1220.1}
  2026-08-02T15:45:11.540671786Z {"ts": "2026-08-02T15:45:11.540206+00:00", "event": "tick", "count": 246, "uptime_s": 1225.1}
```

Every line carries two timestamps, and they come from two different writers.
The outer RFC3339Nano stamp is the platform's, and it marks the moment when the
platform captured the line. The inner `ts` is the worker's own, from `emit`.
The two agree to the millisecond, so `emit` sent each line out at once. Those
lines are also readable while the worker still runs, which is the whole purpose
of `flush=True`.

### The proof of progress

An HTTP recipe proves itself with `curl`. A worker has no URL, so count its
work instead:

```text
get_runtime_log(..., channel="prod", command="grep -c '\"event\": \"tick\"' app.log")
→ ...
  247
```

Call the same command again some minutes later, and the number is larger. Two
numbers and the time between them also give you the rate. This is how you prove
that a worker works, and it is the one check to give a user.

## When it goes wrong

### A fatal setup error stops the deploy

The transcript below comes from a deliberate fault. `set_env` wrote
`TICK_SECONDS=0` on the channel, and the channel got a new deploy. `setup`
refuses a value that is not positive, so the worker stopped before the ready
file existed:

```text
[2026-08-02T22:01:14+00:00] health_check container=b8e0c4721410b968005cbffeedba5568ef0fe867220c66c91c9ae90e01ece098 port=3000 timeout=120.0s
[2026-08-02T22:01:14+00:00] [container] [xhost] starting launch.sh (XHOST_HTTP_PORT=3000) ...
[2026-08-02T22:01:14+00:00] [container] Traceback (most recent call last):
[2026-08-02T22:01:15+00:00] [container]   File "/app/worker.py", line 69, in <module>
[2026-08-02T22:01:15+00:00] [container]     main()
[2026-08-02T22:01:15+00:00] [container]     ~~~~^^
[2026-08-02T22:01:15+00:00] [container]   File "/app/worker.py", line 45, in main
[2026-08-02T22:01:15+00:00] [container]     interval, state = setup()
[2026-08-02T22:01:15+00:00] [container]                       ~~~~~^^
[2026-08-02T22:01:15+00:00] [container]   File "/app/worker.py", line 30, in setup
[2026-08-02T22:01:15+00:00] [container]     raise ValueError(f"TICK_SECONDS must be positive, not {interval}")
[2026-08-02T22:01:15+00:00] [container] ValueError: TICK_SECONDS must be positive, not 0.0
[2026-08-02T22:01:16+00:00] boot failed — removing new container b8e0c4721410b968005cbffeedba5568ef0fe867220c66c91c9ae90e01ece098; previous container keeps serving its own image — this failed deploy did not change its content
[2026-08-02T22:01:17+00:00] deploy failed (host agent): remote agent error (health_check_error): container xhost-26a55c60-59489f61-00000003 exited during boot (exit code 1)
[2026-08-02T22:01:17+00:00] pre-deploy DB snapshot 69701ced-ea7f-4655-b070-0f56e82d4684 was taken before this failure; if the failed deploy changed the database, restore_channel_db can revert to it
```

The real log carries that traceback three times, because the platform started
the container again after each exit. The transcript above shows it one time.
The channel got other deploys between the earlier captures in this recipe and
this one, so the container number is higher here.

Read four facts in that log.

**The error came before the ready file.** This is the central lesson of the
recipe, on a real deploy. The worker never reached `Path(...).touch()`, so no
file existed, and the health check certified nothing. A worker that creates the
file first would pass this deploy, and it would then die and restart without
end.

**`previous container keeps serving its own image`.** The platform removed the
new container and kept the old one. The `app` template bakes your code into a
new image at each deploy, so the container that survives still runs your
previous code. A user of your app sees no change from a failed deploy.

**`exited during boot (exit code 1)`.** The failure came in three seconds, and
not after the 120-second health window. The platform tests that the container
is alive before it tests the ready file, by design. A file that a dead process
left behind must not certify that process.

**`pre-deploy DB snapshot ... was taken before this failure`.** Every non-static
channel gets a database, and the platform takes a snapshot of it before each
deploy. This worker uses no database, so the line changes nothing here. A
worker that uses one can go back to that snapshot with `restore_channel_db`.

### The crashed container is readable, and the old one still runs

Call `get_runtime_log` with no `command` after a failed deploy. The header
describes the dead container:

```text
get_runtime_log(app_name="recipe-worker", channel="prod")
→ container #3 (xhost-26a55c60-59489f61-00000003) — exited
  exit code: 1
  restarts: 4
  started: 2026-08-02T22:01:16.394792105Z
  finished: 2026-08-02T22:01:16.659083603Z
  source: archive (this container was replaced or removed; its log was saved)
  readable containers: #1, #2, #3 (pass container_index to read an older one)
  no command given — pass one (e.g. "tail -n 200 app.log") to read the log itself
```

`source: archive` tells you that the platform removed the container and kept
its log. `readable containers: #1, #2, #3` names the three logs that you can
read now. `container_index` selects one of them, and both reads below use it:

```text
get_runtime_log(..., command="grep -c ValueError app.log", container_index=3)
→ ...
  8

get_runtime_log(..., command="tail -n 3 app.log", container_index=2)
→ container #2 (xhost-26a55c60-59489f61-00000002) — running
  ...
  2026-08-02T22:02:27.584851679Z {"ts": "2026-08-02T22:02:27.584387+00:00", "event": "tick", "count": 3772, "uptime_s": 18856.5}
  2026-08-02T22:02:32.585156823Z {"ts": "2026-08-02T22:02:32.584780+00:00", "event": "tick", "count": 3773, "uptime_s": 18861.5}
  2026-08-02T22:02:37.585645310Z {"ts": "2026-08-02T22:02:37.585294+00:00", "event": "tick", "count": 3774, "uptime_s": 18866.5}
```

That pair of reads shows the platform's failure behaviour.

**The dead container holds eight lines with `ValueError`.** Each attempt writes
two such lines: the `raise` line of the traceback, and the final exception
line. The archive therefore holds four attempts, and the deploy log shows
three. The platform copies no more container lines into the deploy log after
the failure, and the archive keeps every line.

**Container #2 still writes its ticks.** Its three lines above carry the stamp
22:02, and the deploy failed at 22:01. The failed deploy stopped nothing. A
deploy that fails at the health check leaves the previous container in place,
and that container keeps its state and its log. Only the next successful deploy
replaces it.

### The ready file at the top of `launch.sh`

A `launch.sh` that creates `$XHOST_READY_FILE` before it starts the worker
always passes the health check. The channel then reports healthy and does no
work. `worker.py` states the reason at the `Path(...).touch()` call: `setup` can
fail, and that failure must fail the deploy. The file is the signal that the
worker is able to work. Create it only when that is true.

### The health window times out

No deploy in this recipe failed this way, so the message below is an example
with a short container id. It names both signals that the health check accepts:

```text
health check failed for container ...: no 2xx/3xx response at
GET / on port 3000 and no readiness file created at $XHOST_READY_FILE
within 120s
```

For a worker the first half is always true, and you cannot stop the HTTP probe.
The message therefore tells you one thing: your worker did not create
`$XHOST_READY_FILE`. Two causes are usual. The worker blocks on slow work
before it creates the file. Or the worker writes a file with a name of its own,
in place of the name in `$XHOST_READY_FILE`.

### The worker returns, or `launch.sh` has no `exec`

If `main` returns, the process ends and the container exits with the code 0.
The platform starts the container again, because the restart policy is
`unless-stopped`. Inside the health window that exit fails the deploy, with the
same `exited during boot` message as a crash. A worker loop therefore never
ends, and a worker with no work left must wait, not return.

A `launch.sh` without `exec` adds a second fault. The shell stays PID 1, and it
discards the stop signals. Every stop of that container then takes the full
stop timeout.

### A crash loop after the deploy passed

The restart policy is `unless-stopped`, so a worker that dies one hour after a
successful deploy starts again. The deploy stays successful, the channel stays
healthy, and nothing tells you. The `restarts:` count in the status header is
where you see it. Read the header, wait some minutes, and read it again. A
count that grows between the two reads is a crash loop.

### The kernel stopped the container

The kernel stops a worker that asks for too much memory. The process writes
nothing before it stops, so the log looks the same as for a crash. The
status header separates the two. When the kernel stopped the container, the
exit-code line carries the note `(out of memory — the container hit its memory
limit)`. A plain `exit code: 1` with no note is an exception in your own code.

### The log is silent

Two different causes give an empty log.

**Buffered stdout.** Python buffers stdout in blocks when stdout is not a
terminal. A worker without a flush shows nothing for a long time, and then a
block of lines at once. `emit` uses `print(..., flush=True)` for this reason.
For code that you do not own, `set_env(app_name, key="PYTHONUNBUFFERED",
value="1")` gives the same result.

**A log file inside the container.** The platform captures the stdout and the
stderr of your process, and it merges the two. A worker that writes its output
to a file in the container writes nothing that `get_runtime_log` can read. That
file also goes away with the container at the next deploy. Write to stdout.

### The HTTPS hostname returns 502

This is correct, and it is not a fault:

```bash
$ curl -sS -o /dev/null -w '%{http_code}\n' https://recipe-worker-docs.xhostd.app/
502
```

The channel gets `recipe-worker-docs.xhostd.app`, and Caddy routes that
hostname. No process listens on the health port, so the proxy has no upstream.
A 404 from the hostname is different: it tells you that the channel has no
route, because the deploy did not reach `caddy ensure_route`.

To get a worker **and** a URL, serve HTTP on `$XHOST_HTTP_PORT` from the same
container, next to the loop. The platform permits this, and the HTTP arm of the
health check then passes too.

## See also

- [Best known methods](https://docs.xhostd.com/guides/bkm) — how to read a
  runtime log, the health-check checklist, plan budgets, and quota errors.
- [Recipe: Python API on the app template](https://docs.xhostd.com/guides/recipes-app-python)
  — the same template with HTTP, `install.sh` and dependencies.
- [Recipe: a raw TCP service on a public port](https://docs.xhostd.com/guides/recipes-port-forwarding)
  — the other recipe with no URL. It uses the same ready-file signal, and it
  adds a public `host:port`.
- [Recipe: Postgres](https://docs.xhostd.com/guides/recipes-postgres) — where
  the durable state of a worker belongs.
- [Recipe: file uploads](https://docs.xhostd.com/guides/recipes-blob) — where
  the durable files of a worker belong.
