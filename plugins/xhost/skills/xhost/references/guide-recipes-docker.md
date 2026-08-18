# Docker recipe: your own Dockerfile

This recipe deploys a real app from a `Dockerfile` that you write yourself.
This guide is one half of a pair. Both guides describe the same app, and they
divide the subject between them.

- **This guide owns the `docker` template.** It tells you how to write the
  Dockerfile. It covers the warm base images, the image size cap, and the
  content of `CMD`. It also gives the rule that environment variables exist
  at run time only. It shows `Dockerfile`, `requirements.txt` and `app.py`.
- **[The Postgres recipe](https://docs.xhostd.com/guides/recipes-postgres)
  owns the data layer.** It covers `DATABASE_URL`, the psycopg driver
  mismatch, the alembic migrations at container start, and the automatic
  snapshots before a deploy. It shows `alembic.ini` and all the files under
  `migrations/`.

Read this guide first if the `docker` template is new to you. Read the other
guide first if your app builds correctly and the problem is the database.

## What you get

You get a FastAPI service. Your own Dockerfile builds it, and it connects to
the Postgres database that xhost gives the channel. It serves a JSON health
response at `/`. It lists the notes at `GET /notes`. It makes a note at
`POST /notes`. It marks one note done at `POST /notes/{id}/done`.

Alembic migrations create the schema and change it, and they run at container
start. The first boot of a new app thus makes its own tables. A later deploy
with a new migration applies that migration. Neither step is manual. The
build takes two seconds when the layer cache is warm.

The worked example is live at
[recipe-docker-pg-docs.xhostd.com](https://recipe-docker-pg-docs.xhostd.com/).
Read it, but do not write to it. It is a real database behind a real write
API. The note that it lists is the note that this guide's transcript made.
If you obey the recipe on your own account, you get the same service under
your own name.

## The files

The app is `recipe-docker-pg`, and its template is `docker`. This guide shows
three of its eight files.
[The Postgres recipe](https://docs.xhostd.com/guides/recipes-postgres) shows
`alembic.ini`, `migrations/env.py`, `migrations/script.py.mako` and the two
migrations in `migrations/versions/` in full. Those files are the data layer,
not the build. All eight files ship in one commit. The two guides divide the
*prose* only, never the deploy.

### Dockerfile

The `docker` template builds the `Dockerfile` at your repo root on every
deploy. It then runs that image with the image's own `ENTRYPOINT` and `CMD`.
The platform makes no file for you, and it puts nothing into the build.

```dockerfile
# python:3.13-slim is a platform warm base — its layers are exempt from
# the per-plan charged image size, and it is already on the cell so the
# build starts instantly.
FROM python:3.13-slim

WORKDIR /app

# uv resolves and installs far faster than pip. Pinned to an exact
# version tag — append @sha256:<digest> if you need it immutable.
COPY --from=ghcr.io/astral-sh/uv:0.5.14 /uv /usr/local/bin/uv

COPY requirements.txt .
RUN uv pip install --system --no-cache -r requirements.txt

COPY . .

# Migrations run in the START command, never at build time. The build has
# no DATABASE_URL at all — xhost injects env at run time only, never as
# build args — so a build-time migration cannot work even in principle.
CMD ["sh", "-c", "alembic upgrade head && exec uvicorn app:app --host 0.0.0.0 --port $XHOST_HTTP_PORT"]
```

Four parts of that file do real work.

**The base is a warm base.** Every cell already holds these six images, and
the platform does not charge for their layers:

| Warm base |
|---|
| `node:22-slim` |
| `node:24-slim` |
| `python:3.11-slim` |
| `python:3.12-slim` |
| `python:3.13-slim` |
| `debian:trixie-slim` |

If you build `FROM` one of them, the build starts with no pull. Only the
layers that *you* add count against your plan's cap. The platform charges any
other base in full.

**`COPY requirements.txt .` comes before `COPY . .`.** Docker then repeats
the dependency step only when `requirements.txt` changes. If you copy your
whole tree first, every source edit makes that layer invalid. A two-second
deploy then becomes a two-minute deploy.

**xhost injects environment variables at run time only, never as build
args.** There is no `--build-arg` path, and the build can read no secret
store. Your `DATABASE_URL`, your `S3_*` credentials and every variable that
you set appear in one place: the container's environment at container start.
That is why `alembic upgrade head` is in `CMD` and not in a `RUN` step. A
migration at build time has no database to connect to.

**`CMD` uses the `sh -c` form on purpose.** The shell expands
`$XHOST_HTTP_PORT`. The `exec` replaces the shell with uvicorn, so the server
gets the stop signals directly. Your app needs both details. See
[When it goes wrong](#when-it-goes-wrong).

### requirements.txt

This file pins every version exactly. Each later deploy thus builds the same
image that you tested.

```text
fastapi==0.115.6
uvicorn==0.34.0
sqlalchemy==2.0.36
alembic==1.14.0
psycopg[binary]==3.2.3
```

`psycopg[binary]` is psycopg 3. It is not a replacement for `psycopg2`, which
is the driver name that SQLAlchemy selects by default. That mismatch is the
most frequent failure on this platform.
[The Postgres recipe](https://docs.xhostd.com/guides/recipes-postgres) tells
you how to correct it.

### app.py

```python
import os

from fastapi import FastAPI
from pydantic import BaseModel
from sqlalchemy import create_engine, text


def database_url() -> str:
    """xhost injects DATABASE_URL with the bare ``postgresql://`` scheme.

    SQLAlchemy maps that scheme to psycopg2, which we do not ship. Name
    the driver explicitly so it loads psycopg 3 instead.
    """
    return os.environ["DATABASE_URL"].replace(
        "postgresql://", "postgresql+psycopg://", 1
    )


# pool_pre_ping discards connections the database closed while idle,
# which is what otherwise surfaces as a stray 500 after a quiet period.
engine = create_engine(database_url(), pool_pre_ping=True)

app = FastAPI()


class NoteIn(BaseModel):
    body: str


# The health check probes GET / and needs a 2xx, so / must not 404 —
# unless the app creates the file named by $XHOST_READY_FILE instead.
@app.get("/")
def root():
    with engine.connect() as conn:
        row = conn.execute(
            text("SELECT count(*) AS total, count(*) FILTER (WHERE done) AS done FROM notes")
        ).mappings().one()
    return {"ok": True, "notes": row["total"], "done": row["done"]}


@app.get("/notes")
def list_notes():
    with engine.connect() as conn:
        rows = conn.execute(
            text("SELECT id, body, done, created_at FROM notes ORDER BY id")
        ).mappings().all()
    return {"notes": [dict(r) for r in rows]}


@app.post("/notes")
def create_note(note: NoteIn):
    with engine.begin() as conn:
        new_id = conn.execute(
            text("INSERT INTO notes (body) VALUES (:body) RETURNING id"),
            {"body": note.body},
        ).scalar_one()
    return {"id": new_id}


@app.post("/notes/{note_id}/done")
def mark_done(note_id: int):
    with engine.begin() as conn:
        updated = conn.execute(
            text("UPDATE notes SET done = true WHERE id = :id"), {"id": note_id}
        ).rowcount
    return {"updated": updated}
```

Note the route at `/`. The deploy's health check probes `GET /` on the port
that `$XHOST_HTTP_PORT` names, and the value is `3000`. The probe needs a 2xx
or 3xx answer within 120 seconds. An API with all its routes under `/api`
fails its deploy, although the process runs correctly. The one alternative is
the file that `$XHOST_READY_FILE` names, which the probe also accepts.

## The deploy

Every app owns a git repo, and **`git push` and then `deploy` is the standard
path**. A push sends only the diff, which keeps the second, tenth and
hundredth edit cheap. Git transfers the few lines that changed. A tool call
instead carries the full text of every file. Use `commit_files` in one
situation only: git is not available on the machine where you work.

Both paths keep two acts separate. **A push stores your code, but it does not
deploy your code.** `deploy` is its own explicit call, and that is the point:
you name the commit that goes live.

Do these four steps in order. All eight files ship in the one push.

**1. Create the app.** This call makes the `prod` channel, its hostname, its
git repo and its Postgres database.

```text
create_app(name="recipe-docker-pg", template="docker")
→ {"id": "d2ae0f30-36f0-443f-95d1-d937a7bbb676",
   "name": "recipe-docker-pg",
   "template": "docker",
   "repo_url": "https://git.xhostd.com/docs/recipe-docker-pg.git",
   "channels": [{"id": "50bdd958-d4e7-41db-8adb-9a49cd8966fd",
                 "name": "prod",
                 "hostname": "recipe-docker-pg-docs.xhostd.com",
                 "current_sha": null}], ...}
```

Keep the app id and the `prod` channel id, because every later call needs one
or both. `repo_url` is the repo for your push. `get_app` returns that URL
again if you lose it.

**2. Make a credential.** One token is your git password, your Postgres
password and your platform API bearer. It is valid for 30 days.

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

Put the token in the **password** field of the remote URL. That detail is the
important one:

```text
https://<username>:<token>@git.xhostd.com/<username>/<app>.git
```

**3. Clone the repo, then commit and push.** A new xhost repo is empty, and
git tells you so. This warning is normal, not a fault:

```bash
$ git clone https://docs:$XHOST_TOKEN@git.xhostd.com/docs/recipe-docker-pg.git
Cloning into 'recipe-docker-pg'...
warning: You appear to have cloned an empty repository.
```

Write the eight files into that directory. Then commit and push them:

```bash
$ cd recipe-docker-pg
$ git add -A
$ git commit -m "notes API on the docker template"
$ git push origin master
```

Never put the token in a file that you commit, or in text that you paste.
`$XHOST_TOKEN` above holds the value from `get_credentials`. A remote URL
with a real token in it goes into `.git/config`, into your shell history and
into the output of every `git remote -v`.

**4. Deploy the branch.**

```text
deploy(app_id="d2ae0f30-36f0-443f-95d1-d937a7bbb676",
       channel_id="50bdd958-d4e7-41db-8adb-9a49cd8966fd",
       ref="master")
→ {"deploy_id": "abf7a32e-394d-4646-b774-0c12c1c3f046",
   "channel_id": "50bdd958-d4e7-41db-8adb-9a49cd8966fd",
   "status": "queued"}
```

`ref` is a branch name, and xhostd resolves it to that branch's current head.
After a push you thus do not need the sha. `sha` is also valid when you want
an exact commit, and `sha` wins if you give both.

The deploy runs asynchronously. Follow it with
`get_deploy_log(app_id=..., channel_id=..., deploy_id=...)`. That call
returns the log as plain text. The log ends with `deploy success`, or with a
line that names the failure.

## Verify it

This is the deploy above, `abf7a32e-394d-4646-b774-0c12c1c3f046`. The
transcript shows short ids, and it omits the buildkit lines that teach
nothing.

```text
[2026-07-31T18:38:41+00:00] deploy begin id=abf7a32e-... channel=50bdd958-... sha=aacda699...
[2026-07-31T18:38:41+00:00] git_sync ok: synced app=d2ae0f30-... channel=50bdd958-... sha=aacda699...
[2026-07-31T18:38:41+00:00] [build] start sha=aacda69971a19f0b7e37e613d21b0b361c98c1fa
[2026-07-31T18:38:41+00:00] [build] queued 0s, starting
[2026-07-31T18:38:42+00:00] [build] #7 [stage-0 1/6] FROM docker.io/library/python:3.13-slim@sha256:6771159cd4fa...
[2026-07-31T18:38:42+00:00] [build] #8 [stage-0 3/6] COPY --from=ghcr.io/astral-sh/uv:0.5.14 /uv /usr/local/bin/uv
[2026-07-31T18:38:42+00:00] [build] #8 CACHED
[2026-07-31T18:38:42+00:00] [build] #11 [stage-0 5/6] RUN uv pip install --system --no-cache -r requirements.txt
[2026-07-31T18:38:42+00:00] [build] #11 CACHED
[2026-07-31T18:38:42+00:00] [build] #12 [stage-0 6/6] COPY . .
[2026-07-31T18:38:42+00:00] [build] #12 DONE 0.0s
[2026-07-31T18:38:43+00:00] [build] finished in 2s
[2026-07-31T18:38:43+00:00] [build] queue wait 0s, build 2s
[2026-07-31T18:38:43+00:00] [build] image 262.09 MB total, 94.63 MB charged — base python:3.13-slim exempt
[2026-07-31T18:38:45+00:00] channel snapshot saved: 0.00 MB
[2026-07-31T18:38:46+00:00] start_static_site template=docker
[2026-07-31T18:38:46+00:00] health_check container=6f0c1f9382cb... port=3000 timeout=120.0s
[2026-07-31T18:38:50+00:00] [container] INFO  [alembic.runtime.migration] Running upgrade  -> 0001, create notes
[2026-07-31T18:38:50+00:00] [container] INFO  [alembic.runtime.migration] Running upgrade 0001 -> 0002, add done flag to notes
[2026-07-31T18:38:54+00:00] [container] INFO:     Uvicorn running on http://0.0.0.0:3000 (Press CTRL+C to quit)
[2026-07-31T18:38:54+00:00] health_check ok
[2026-07-31T18:38:54+00:00] [container] INFO:     10.77.1.5:45294 - "GET / HTTP/1.1" 200 OK
[2026-07-31T18:38:54+00:00] caddy ensure_route hostname=recipe-docker-pg-docs.xhostd.com upstream=10.77.1.5:32044
[2026-07-31T18:38:55+00:00] deploy success
```

Read these six lines on every deploy.

**`#11 CACHED`, and `build 2s`.** Docker used the cached dependency layer
again, and it did not repeat the step. Only `#12 COPY . .`, your source, ran.
That is the result of the `COPY` order. Docker copies `requirements.txt` on
its own, so it builds the dependency layer again only when your dependencies
change. A deploy that changes the source alone costs about two seconds.

**`image 262.09 MB total, 94.63 MB charged — base python:3.13-slim exempt`.**
This is the most useful line for a `docker` user. *Total* is the size of the
image on disk. *Charged* is the size that counts against your plan's image
size cap. The platform subtracts the layers of the largest warm base under
your image. Here that base is the ~167 MB difference, and the platform
charges you for the 94.63 MB that you added. It subtracts one base only, and
only if your image is truly built on that base.

Each plan has its own cap: basic 512 MiB, builder 2 GiB, indie 4 GiB, pro
12 GiB.

**`channel snapshot saved: 0.00 MB`.** Every non-static deploy saves a
snapshot of the channel's Postgres schema before the new container starts.
You do not ask for the snapshot, and you cannot forget it. This snapshot
rounds to 0.00 MB because the database is still empty on the app's first
deploy. A snapshot holds the state *before* its own deploy, and this database
holds nothing at that point. See
[the Postgres recipe](https://docs.xhostd.com/guides/recipes-postgres) for
the calls that list and restore a snapshot.

**The two `Running upgrade` lines.** These lines are `alembic upgrade head`
inside the container. It makes the schema from nothing on the app's first
boot: `-> 0001`, then `0001 -> 0002`, in that order. A deploy that adds a
third migration prints one more line, and nothing else changes. If you see
the alembic banner with no `Running upgrade` line, the database is already at
head and alembic had no work.

**`health_check ... port=3000 timeout=120.0s`, then `health_check ok` eight
seconds later.** In those eight seconds the migrations run, and then uvicorn
binds its port. The probe gets its first 2xx after that. The 120-second
window gives a migration the time to finish.

**`caddy ensure_route`.** xhostd points the hostname at the new container
only after the health check passes. On a redeploy, the line that retires the
previous container, `stop_and_remove old container=...`, comes after that.
That order is the reason why a redeploy loses no requests.

Then the two-line proof, against the live demo:

```bash
$ curl -sS https://recipe-docker-pg-docs.xhostd.com/
{"ok":true,"notes":1,"done":0}

$ curl -sS https://recipe-docker-pg-docs.xhostd.com/notes
{"notes":[{"id":1,"body":"first note from the recipe","done":false,"created_at":"2026-07-31T18:39:10.159203+00:00"}]}
```

`POST /notes` wrote that note after the deploy. The
[Postgres recipe](https://docs.xhostd.com/guides/recipes-postgres) shows the
call, and what then happened to the row.

To learn whether the app is alive, call `get_runtime_log` with **no**
`command`:

```text
get_runtime_log(app_id="d2ae0f30-36f0-443f-95d1-d937a7bbb676", channel="prod")
```

Without a `command`, the call starts no container on the host. It returns the
status header alone. The header gives the process state, the exit code if the
process stopped, and whether the kernel OOM-killed it. It also gives the
restart count and the container indices that you can still read. This call is
the fastest check for a crash.

Those indices are useful after a redeploy. The platform archives the previous
container's log, so you can still read the output of the version that
crashed. Give its `container_index` to `get_runtime_log`.

## When it goes wrong

### You expected build args to carry secrets or `DATABASE_URL`

They cannot. xhost injects the environment at run time only. There is no
`--build-arg` and no build-time secret mount. The build cannot read
`DATABASE_URL`, because that value does not exist at build time. The symptom
is a `RUN` step that fails on an absent variable, or an `os.environ[...]`
`KeyError` in the build. Move the work into `CMD`.

A second rule is as important. A build can never read a secret, so never put
a secret into the image yourself.

### The image is over your plan's cap

The deploy fails at the build step. The message names the charged size, the
cap and your plan, and the platform removes the image. Read the
`total`/`charged` line first. If `charged` is near `total`, your base is not
a warm base, and the platform charges you for all of it. Change `FROM` to one
of the six warm bases above, which usually corrects the whole problem. If
`charged` is truly large, your runtime image holds build tools. Use a
multi-stage build that copies only the artifact.

### `CMD` does not `exec` the final process

The form `CMD ["sh", "-c", "alembic upgrade head && uvicorn ..."]` has no
`exec`. The shell then stays alive as PID 1, and uvicorn is its child. The
stop signals go to the shell, and the shell does not send them to uvicorn.
The platform thus kills the server instead of a clean stop, and every
redeploy waits for the full stop timeout. Put `exec` before the last command
in the chain.

### `GET /` returns 404, so the deploy fails even though the app started

The health check needs a 2xx or 3xx answer from `GET /` on the health port
within 120 seconds. As an alternative, the file that `$XHOST_READY_FILE`
names must exist. An app that serves `/api/...` only, and makes no ready
file, fails the deploy. Its own logs still show a correct start. Add a root
route, and a JSON `{"ok": true}` is enough. This recipe's deploy succeeded,
so its log does not hold the message below. The message has this shape, with
a short container id:
`health check failed for container ...: no 2xx/3xx response at GET / on
port 3000 and no readiness file created at $XHOST_READY_FILE within 120s`.

### The server listens on a hardcoded port

Bind the port that `$XHOST_HTTP_PORT` names, not a literal port. The platform
selects the value, and the health check probes that port only. The probe
cannot see a server on any other port.

This failure is easy to miss. `CMD` uses the `sh -c` form for one reason: the
shell expands the variable. The pure exec form —

```dockerfile
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "$XHOST_HTTP_PORT"]
```

— expands nothing. uvicorn gets the literal 16-character string
`$XHOST_HTTP_PORT`, and it stops on that value. Use `sh -c`, or read the
variable in your own code.

### 404 and 502 from the hostname mean different things

A channel that exists but has no route returns **404** on its hostname. A
channel with a route but no live server returns **502**. Both codes help you.
A 404 tells you that the deploy did not reach `caddy ensure_route`. A 502
tells you that the deploy reached it, but the container does not serve.
