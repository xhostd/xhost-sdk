# Postgres recipe: migrations and snapshots

This recipe shows the data layer of a real app. It tells you how to connect
to the Postgres database that xhostd gives you. It tells you how to run
alembic migrations safely. It tells you how to use the snapshots that the
platform takes before every deploy. This guide is one half of a pair. Both
guides describe the same app, and they divide the subject between them.

- **This guide owns the data layer.** It covers `DATABASE_URL`, the psycopg
  driver mismatch, the migrations in the start command, and the automatic
  snapshots before a deploy. It shows `alembic.ini`, `migrations/env.py`,
  `migrations/script.py.mako` and both files under `migrations/versions/`.
- **[The Docker recipe](https://docs.xhostd.com/guides/recipes-docker) owns
  the build.** It covers the Dockerfile, the warm base images, the image
  size cap, the `CMD` rules, and the health check. It shows `Dockerfile`,
  `requirements.txt` and `app.py`.

Nothing here is specific to the `docker` template. The same rules apply to
the `app` template, where the start command is `launch.sh` instead of `CMD`.

## What you get

Each non-static channel gets its own Postgres database. The platform makes that
database when it makes the channel. Your container reads the connection details
from an injected `DATABASE_URL`. A static channel has no database, so it gets no
`DATABASE_URL`.

The container also carries `DATABASE_URL_READONLY`: the same database, a
second role that can `SELECT` from every table in `public` and run no write.
Use it for a query surface you expose to visitors. Writes and migrations use
`DATABASE_URL`. Data the read-only role must not see goes in a schema you
create (`CREATE SCHEMA private`). The Postgres reference page has the whole
contract.

Schema changes ship as ordinary alembic migrations, and they run at container
start. The first boot of a new app applies every migration in order against
an empty database. A later deploy applies only the migrations that are new.
There is no manual step and no separate migration job.

Before each of those deploys, the platform saves a snapshot of the database.
The snapshot gives you a one-call undo if the deploy is wrong.

The worked example is live at
[recipe-docker-pg-docs.xhostd.app](https://recipe-docker-pg-docs.xhostd.app/).
Read it, but do not write to it. It is a real database behind a real write
API. The one note that it lists is the note that this guide restored it to.
If you obey the recipe on your own account, you get the same app under your
own name.

## The files

The app is `recipe-docker-pg`. This guide shows the five files of its data
layer. [The Docker recipe](https://docs.xhostd.com/guides/recipes-docker)
shows `Dockerfile`, `requirements.txt` and `app.py` in full. All eight files
ship in one commit. The two guides divide the prose only, never the deploy.

### How the app connects

xhostd sets one value for the database connection, and you set nothing. The
value is `DATABASE_URL`. Your channel owns a whole database, and your data is
in the standard `public` schema of that database. Your SQL therefore uses
plain table names — `notes`, not `someschema.notes`.

xhostd gives you `DATABASE_URL` with the bare `postgresql://` scheme:

```text
postgresql://<role>:<password>@<host>:<port>/<database>
```

Do not use that scheme without a change from Python. SQLAlchemy maps
`postgresql://` to **psycopg2**, but this app ships **psycopg 3**
(`psycopg[binary]`). The correction is one line. It appears twice in this
app: once in `app.py`, and once in `migrations/env.py`.

```python
def database_url() -> str:
    """xhostd injects DATABASE_URL with the bare ``postgresql://`` scheme.

    SQLAlchemy maps that scheme to psycopg2, which we do not ship. Name
    the driver explicitly so it loads psycopg 3 instead.
    """
    return os.environ["DATABASE_URL"].replace(
        "postgresql://", "postgresql+psycopg://", 1
    )
```

If you do not make this correction, Python reports
`ModuleNotFoundError: No module named 'psycopg2'`. See
[When it goes wrong](#when-it-goes-wrong).

### alembic.ini

```ini
[alembic]
script_location = migrations

# sqlalchemy.url is deliberately absent: migrations/env.py reads
# DATABASE_URL from the environment instead, so no connection string is
# ever committed to the repo.

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARNING
handlers = console
qualname =

[logger_sqlalchemy]
level = WARNING
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stdout,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
```

The absent `sqlalchemy.url` is the point. xhostd injects your credentials, and
it rotates them independently of your code. Never put them in a file that you
commit.

Keep the log configuration blocks, but note that they do nothing until
`env.py` applies them. The next file shows how. After `env.py` applies them,
alembic logs at `INFO`. The line
`Running upgrade 0001 -> 0002, add done flag to notes` then goes to stdout.
The deploy log and `get_runtime_log` both read stdout.

### migrations/env.py

```python
import os
from logging.config import fileConfig

from alembic import context
from sqlalchemy import create_engine


def _url() -> str:
    return os.environ["DATABASE_URL"].replace(
        "postgresql://", "postgresql+psycopg://", 1
    )


def run_migrations_online() -> None:
    engine = create_engine(_url())
    with engine.connect() as connection:
        context.configure(connection=connection)
        with context.begin_transaction():
            context.run_migrations()


# alembic.ini's logging blocks stay inert until env.py applies them.
if context.config.config_file_name is not None:
    fileConfig(context.config.config_file_name)

run_migrations_online()
```

This `env.py` is minimal on purpose: no offline mode, no `target_metadata`,
and no autogenerate. Autogenerate compares your models to the live database,
so it needs both of them. A migration that you write by hand needs neither.
A smaller file has fewer failure modes.

Do not remove the `fileConfig` call. No other line reads the log
configuration blocks in `alembic.ini`. Without the call, the configuration
stays inert. Python then uses its last-resort handler at `WARNING`, and it
discards every `Running upgrade` line before the line reaches stdout. The
migration still runs. You lose the only proof in the deploy log that it ran,
and that proof is what you want when a deploy fails.

### migrations/script.py.mako

Alembic uses this template when you run `alembic revision`. Alembic needs the
file, and it fails without it. The content below is the standard content.

```text
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}

"""
from alembic import op
import sqlalchemy as sa
${imports if imports else ""}

revision = ${repr(up_revision)}
down_revision = ${repr(down_revision)}
branch_labels = ${repr(branch_labels)}
depends_on = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}
```

### migrations/versions/0001_create_notes.py

This is the first migration. It creates the table from nothing. The database
of each new channel starts in that empty state.

```python
"""create notes

Revision ID: 0001
Revises:

"""
import sqlalchemy as sa
from alembic import op

revision = "0001"
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        "notes",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("body", sa.Text, nullable=False),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            server_default=sa.func.now(),
            nullable=False,
        ),
    )


def downgrade() -> None:
    op.drop_table("notes")
```

### migrations/versions/0002_add_note_done.py

This is the second migration. It adds a column to a table that can already
hold rows. It must be correct against the empty database of a first boot. It
must also be correct against a table with a year of production data.

```python
"""add done flag to notes

Revision ID: 0002
Revises: 0001

"""
import sqlalchemy as sa
from alembic import op

revision = "0002"
down_revision = "0001"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # server_default plus nullable=False so the rows already in the table
    # get a value without a separate backfill step.
    op.add_column(
        "notes",
        sa.Column(
            "done", sa.Boolean(), server_default=sa.false(), nullable=False
        ),
    )


def downgrade() -> None:
    op.drop_column("notes", "done")
```

Learn the pattern of `server_default=sa.false()` with `nullable=False`.
Postgres fills the rows that exist from the default as part of the
`ADD COLUMN`. There is thus no separate backfill step, and the column is
never nullable. If you remove the `server_default`, the same migration still
runs in development against an empty table. It then fails at the first table
that holds rows. See [When it goes wrong](#when-it-goes-wrong). That default
is the reason why every note from this app reports `"done":false`, the first
note included.

Both migrations are in the app's first and only commit, and that is the usual
case. A migration ships in the same commit as the code that needs its column.
If you commit `app.py` without `0002`, you deploy an app whose every query
names a column that the database does not have.

A `downgrade()` costs one line here, so write one. But it is not the tool for
an emergency. The snapshots are that tool. They also work when the migration
leaves the schema in a state that `downgrade()` cannot reverse.

## The deploy

The whole app ships in one commit and one deploy: all eight files and both
migrations. **`git push` and then `deploy` is the standard path.** A push
sends only the diff, and needs no anchor to place it. The next schema change
thus costs a one-file commit and a push, not a tool call carrying anchors or
whole files. `commit_files` is
the fallback for one situation only: git is not available on the machine
where you work.

Both paths keep the same rule. **A push stores your code, but it does not
deploy your code.** `deploy` is a separate and explicit call.
[The Docker recipe](https://docs.xhostd.com/guides/recipes-docker) gives the
full sequence step by step: `create_app`, `get_credentials`, the clone and
the push, then `deploy`. This guide shows only the call that matters here:

```text
deploy(app_name="recipe-docker-pg",
       channel="prod",
       ref="master")
→ {"deploy_id": "abf7a32e-394d-4646-b774-0c12c1c3f046",
   "channel_id": "50bdd958-d4e7-41db-8adb-9a49cd8966fd",
   "status": "queued"}
```

`ref` resolves to the current head of that branch. A push and then a `deploy`
thus needs no sha.

That sequence has no migration step and no migration tool. The migrations
ship because they are in the commit. They run because the container's start
command runs `alembic upgrade head` before it starts the server:

```dockerfile
CMD ["sh", "-c", "alembic upgrade head && exec uvicorn app:app --host 0.0.0.0 --port $XHOST_HTTP_PORT"]
```

The start command is the *only* correct place for `alembic upgrade head`.
xhostd injects environment variables at run time only, never as build args. A
build thus has no `DATABASE_URL`, and a migration at build time cannot work.
On the `app` template the same line goes in `launch.sh`, never in
`install.sh`, for the same reason.

## Verify it

Seven lines of that deploy log tell you what the data layer did.

```text
[2026-07-31T18:38:45+00:00] channel snapshot saved: 0.00 MB
[2026-07-31T18:38:46+00:00] health_check container=6f0c1f9382cb... port=3000 timeout=120.0s
[2026-07-31T18:38:50+00:00] [container] INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
[2026-07-31T18:38:50+00:00] [container] INFO  [alembic.runtime.migration] Will assume transactional DDL.
[2026-07-31T18:38:50+00:00] [container] INFO  [alembic.runtime.migration] Running upgrade  -> 0001, create notes
[2026-07-31T18:38:50+00:00] [container] INFO  [alembic.runtime.migration] Running upgrade 0001 -> 0002, add done flag to notes
[2026-07-31T18:38:54+00:00] health_check ok
```

**`channel snapshot saved`** is automatic. Every non-static deploy saves a
snapshot of the channel's Postgres database *before* the new container starts.
The snapshot is thus the state immediately before that deploy's changes. This
snapshot rounds to 0.00 MB because the database is still empty on the app's
first deploy. The snapshot list below gives the true size: 2159 bytes.

**The two `Running upgrade` lines** show the migrations at work. They run in
revision order against that empty database: `-> 0001` creates the table, then
`0001 -> 0002` adds the column. Alembic prints the `Context impl` and
`Will assume transactional DDL` banner on every run. Those two banner lines
tell you only that alembic started. The `Running upgrade` lines tell you that
the schema changed.

**`health_check ok` comes eight seconds after the probe starts.** In those
eight seconds `alembic upgrade head` runs, and then uvicorn binds its port.
The 120-second health window gives a migration the time to finish. The deploy
becomes healthy the moment the server answers `GET /`.

The schema now exists, so the API works. Write a row through it:

```bash
$ curl -sS -X POST https://recipe-docker-pg-docs.xhostd.app/notes \
    -H 'content-type: application/json' \
    -d '{"body":"first note from the recipe"}'
{"id":1}
```

### The snapshots

If you deploy the same commit again, you get a second snapshot. You also get
a second alembic transcript to compare with the first:

```text
[2026-07-31T18:39:18+00:00] channel snapshot saved: 0.01 MB
[2026-07-31T18:39:24+00:00] [container] INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
[2026-07-31T18:39:24+00:00] [container] INFO  [alembic.runtime.migration] Will assume transactional DDL.
[2026-07-31T18:39:29+00:00] health_check ok
```

The banner is there, but no `Running upgrade` line comes after it. The
database is already at head, so `alembic upgrade head` had no work and
printed no upgrade line. Read a deploy log by that difference. The banner
with a `Running upgrade` line means that the schema changed. The banner alone
means that the schema did not change.

Both snapshots, newest first (`created_at` elided):

```text
list_channel_snapshots(app_name="recipe-docker-pg", channel="prod")
→ [
    {"snapshot_id": "96879c2a-d99f-4a97-ab13-0de58442bd5f",
     "deploy_id": "e39a2b93-d2b0-4c43-a74d-5eefce5a804d",
     "created_at": "..."},
    {"snapshot_id": "3a52f325-b7d3-4960-8105-4e44db6284da",
     "deploy_id": "abf7a32e-394d-4646-b774-0c12c1c3f046",
     "created_at": "..."}
  ]
```

Read the `deploy_id` on each snapshot. A snapshot is the state immediately
**before** that deploy. The older snapshot holds the empty schema, from the
point before the first deploy's migrations. The newer snapshot holds the
table, which then existed and held the note.

### Restore the database

Add a second note, *after* the platform takes that newer snapshot:

```bash
$ curl -sS -X POST https://recipe-docker-pg-docs.xhostd.app/notes \
    -H 'content-type: application/json' \
    -d '{"body":"second note, added after the snapshot"}'
{"id":2}
```

xhostd refuses a restore of **`prod`** unless the app's environment holds
`XHOST_ALLOW_PROD_RESTORE=1`. The refusal reads `prod_restore_blocked`, and
it is the guard at work, not a fault. Keep the guard until you are sure that
you want the restore. A restore saves no snapshot of the state that it
replaces, because the platform saves a snapshot before a deploy only. The
rows that the restore overwrites are thus not in the snapshot list after it.
You set the variable to confirm that you accept this:

```text
set_env(app_name="recipe-docker-pg",
        key="XHOST_ALLOW_PROD_RESTORE", value="1")
```

Then restore the channel to the snapshot from immediately before the second
deploy. That snapshot holds note 1 and not note 2.

```text
restore_channel_db(app_name="recipe-docker-pg",
                   channel="prod",
                   snapshot_id="96879c2a-d99f-4a97-ab13-0de58442bd5f")
```

The restore runs in one transaction on the server. xhostd first empties the
database's `public` schema, then it writes the snapshot's contents into that
schema. The server rolls the whole transaction back if any step fails, so a
failed restore loses nothing. The call returns the channel's Postgres status,
which is `ready` when the restore succeeds. The same `DATABASE_URL` continues
to work with no new deploy. Only the data changes: it is back at the state of
the snapshot.

```bash
$ curl -sS https://recipe-docker-pg-docs.xhostd.app/
{"ok":true,"notes":1,"done":0}

$ curl -sS https://recipe-docker-pg-docs.xhostd.app/notes
{"notes":[{"id":1,"body":"first note from the recipe","done":false,"created_at":"2026-07-31T18:39:10.159203+00:00"}]}
```

Note 2 is gone. Note 1 is back with its original `created_at`, not a new one.
A restore writes the snapshot's rows again, but it does not run your writes
again. The `"done":false` value on the note comes from `0002`'s
`server_default`. The app's `INSERT` names `body` only, so Postgres supplies
the column's default. The same default fills the column for the rows already
in the table when the migration runs. A migration without that default fails
in exactly that case.

There is a second guard. xhostd refuses a restore while a deploy on the
channel is queued or in progress. Wait for the deploy to finish, then try
again.

A restore of the database does not restore your code. If you restore to a
state before a migration, also deploy the commit that matches that state.
If you do not, the next container start applies the migration again.

## PostGIS

The database server marks PostGIS trusted, so a migration installs it
with no superuser. Install the extension in the migration that first needs
a geometry column, add the column, and add a GiST index on it in the same
migration. The extension is a one-time cost of about two seconds and
7.5 MB per database.

```python
def upgrade() -> None:
    op.execute("CREATE EXTENSION IF NOT EXISTS postgis")
    op.add_column(
        "places",
        sa.Column("location", Geometry("POINT", srid=4326), nullable=True),
    )
    op.create_index(
        "ix_places_location",
        "places",
        ["location"],
        postgresql_using="gist",
    )
```

`Geometry` comes from the `geoalchemy2` package; add it to
`requirements.txt` beside `psycopg[binary]`. Raw SQL works the same way:
`ALTER TABLE places ADD COLUMN location geometry(Point, 4326)` and
`CREATE INDEX ix_places_location ON places USING GIST (location)`.

Only `postgis` is installable. `postgis_raster`, `postgis_topology`,
`postgis_sfcgal`, and the tiger geocoder stay untrusted. `spatial_ref_sys`
is read-only for your role, with every EPSG code present. A snapshot
restore, a move, and an export all carry a PostGIS database. The Postgres
reference page has the details.

## When it goes wrong

### `ModuleNotFoundError: No module named 'psycopg2'`

This is the most frequent failure for a Python app on this platform, and it
has one cause. `DATABASE_URL` arrives with the bare `postgresql://` scheme,
and SQLAlchemy maps that scheme to psycopg2. Most projects do not install
psycopg2, because psycopg 3 is the driver that Python projects ship.

The container starts and then stops immediately, so the deploy fails on the
health check. No line in the deploy log mentions the database. Read the
container's own output, not the health-check line alone:

```text
get_runtime_log(app_name="...", channel="prod", command="tail -n 200 app.log")
```

To correct it, name the driver in the URL at each point where you make an
engine:

```python
os.environ["DATABASE_URL"].replace("postgresql://", "postgresql+psycopg://", 1)
```

Do this in `migrations/env.py` and in your application code. If you do it in
one place only, the failure is harder to find: the app serves requests, and
the migration step is the part that stops.

You can instead install `psycopg2-binary` and keep the URL as it is. That
also works, but your dependency tree then holds two Postgres drivers as soon
as another package needs psycopg 3.

### The migration ran at build time, or not at all

`alembic upgrade head` cannot work in a `RUN` step, or in `install.sh` on the
`app` template. xhostd injects the environment at run time only, never as
build args, so a build has no `DATABASE_URL`. The symptom changes with the
way your code reads the variable. You see a `KeyError`, a refused connection,
or a migration that applies to nothing.

Put the command in the start command: `CMD` for `docker`, `launch.sh` for
`app`. Put it before the line that starts your server, and join the two with
`&&`. A failed migration then stops the deploy. Without the `&&`, the server
starts against a stale schema.

### The code shipped without the migration it needs

Every query in `app.py` names the `done` column, and `done` exists because
`0002` creates it. If you commit one without the other, the container starts,
connects, and stops at its first query:

```text
sqlalchemy.exc.ProgrammingError: (psycopg.errors.UndefinedColumn) column "done" does not exist
```

The deploy then fails on the health check, and its log says nothing about a
column. The exception is in the container's own output, which you read with
`get_runtime_log`. Ship a migration in the same commit as the code that needs
it. The code and the schema are then never one deploy apart, and
`alembic upgrade head` in the start command is sufficient. Some changes
cannot travel together, such as a rename or a column drop. For those, first
write code that accepts both shapes. Then make the schema change in a later
deploy, after no live version needs the old shape.

### A `NOT NULL` column with no `server_default`

This recipe's `0002` migration sets a `server_default`, so it does not have
this failure. If you remove that default, Postgres reports:

```text
column "done" of relation "notes" contains null values
```

A `NOT NULL` column that you add to a table with rows fails, because Postgres
has no value for those rows. The migration passes in development against an
empty table. It fails at the first deploy to a channel with real data.

Add `server_default` in the same `op.add_column`, as `0002` does. If the
value can have no default, use three migrations. Add the column as nullable,
backfill it, then set `NOT NULL`. Note that the backfill step can exceed the
health-check window.

### The migration takes longer than the health check allows

The health window is 120 seconds from container start, and your migration
runs inside it. A migration that rewrites a large table can exhaust the
window. A migration that waits for a lock from the old container, which still
runs, can also exhaust it. The deploy then fails on the health check while
the migration is still in progress.

Keep a deploy-time migration short, and do not let it block. Use
`CREATE INDEX CONCURRENTLY` in place of a plain `CREATE INDEX`. Add a column
with a default in place of a rewrite of the rows. Move a long backfill out of
the deploy path. Ship the schema change first, then run the backfill as its
own step after the app is live.

### 404 and 502 from the hostname mean different things

A channel that exists but has no route returns **404** on its hostname. A
channel with a route but no live server returns **502**. Both codes help you.
A 404 tells you that the deploy did not reach `caddy ensure_route`. A 502
tells you that the deploy reached it, but the container does not serve. For a
Python app on this platform, a 502 is often the psycopg2 failure above.
