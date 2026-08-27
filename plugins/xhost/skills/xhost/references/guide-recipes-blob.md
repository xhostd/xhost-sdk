# Blob recipe: uploads on the channel's object store

## What you get

A FastAPI service stores uploaded files in the S3-compatible object store.
xhostd provisions that store for the channel. `POST /files` uploads one file.
`GET /files` lists the files. `GET /files/{key}` downloads one file.
`DELETE /files/{key}` removes one file. You create no bucket, you configure no
credentials and you set no flag. The platform injects five `S3_*` variables
into the container, and boto3 uses them unchanged.

The two reads are public. The two writes need
`Authorization: Bearer <token>`. The app compares that token with a
`WRITE_TOKEN` that you set yourself with
`set_env(app_id, key="WRITE_TOKEN", value=..., secret=True)`.

The split between the reads and the writes is necessary. An app on a public
hostname that accepts files from anyone is an open object store. That store
holds the bytes that your plan's quota counts. If you do not set
`WRITE_TOKEN`, the app refuses every write. Thus a forgotten secret gives you
a locked app, never a public one.

The example app is live at
[recipe-blob-docs.xhostd.app](https://recipe-blob-docs.xhostd.app/). The
public can only read it. The `GET` calls in this guide reach the app. The app
refuses the `POST` and `DELETE` calls, because this guide does not publish its
write token. If you do the recipe on your own account, you get the same
service under your own name, and you hold the only token.

The app runs on the `app` template. Thus `install.sh` and `launch.sh` obey the
contract in
[Recipe: Python API on the app template](https://docs.xhostd.com/guides/recipes-app-python).
This guide adds only the storage layer.

## The five injected variables

Every channel has its own object store. The platform provisions that store
when it creates the channel. When the store becomes `ready`, each deploy of
the channel injects these five variables into the container. Before the store
is `ready`, no deploy injects them. A store that is not yet `ready` does not
stop a deploy: the deploy continues, but the variables are absent.

| Variable | What it holds |
|---|---|
| `S3_ENDPOINT` | This cell's own S3 gateway, an `http://<overlay-ip>:6533` URL on the private WireGuard overlay |
| `S3_BUCKET` | The channel's virtual bucket name — the first label of its hostname, here `recipe-blob-docs` |
| `S3_ACCESS_KEY_ID` | The channel's own scoped key id |
| `S3_SECRET_ACCESS_KEY` | Its secret |
| `S3_REGION` | `us-east-1`, a neutral placeholder |

Read these five facts before you write code.

**Stock SigV4 works. You do not need a botocore `Config`.** The endpoint is
your own cell's gateway on a private overlay network. Thus the URL is plain
`http`, and you configure nothing about TLS, the `addressing_style` or the
signature version. Pass `endpoint_url`, the key pair and the region to
`boto3.client("s3", ...)`. Pass nothing more. A `Config(...)` is the most
common unnecessary edit here. A wrong `Config(...)` — an old
`signature_version`, or a forced `addressing_style` — makes a correct client
fail with a signature error.

**Only code inside the container can reach that endpoint.** It is an overlay
address. Nothing on your laptop can route to it.

**Pass `S3_BUCKET` exactly as the platform gives it to you.** The gateway
compares the bucket name in every request with the channel's own name. Any
other name gets a 404 `NoSuchBucket`, on a read, a write and a copy alike.
Read the value from the environment. Do not hardcode it, and do not reuse
another channel's name.

**`S3_REGION` is a placeholder.** The real upstream region never appears in
your container. Pass the value, because the SDK needs one. Give it no other
meaning.

**Keys are plain and have no prefix.** Upstream, the gateway puts every object
under this channel's own private key prefix. The key `notes.txt` in your code
is `notes.txt` in every list. It cannot collide with another channel's object
of the same name, and it cannot reach one. The gateway refuses access between
channels; this is not only a convention. Keep out of one name space:
`.xhost/`. The platform keeps `.xhost/` for its own deploy snapshot markers.
The gateway refuses a key under `.xhost/` with a 403, and removes the markers
from your lists.

**You enable nothing.** The app record shows a flag named
`external_blob_access_enabled`. That flag is **not** part of this recipe. It
controls whether S3 tools *outside* the platform can reach the store through
the public gateway. Examples are `aws s3` on your laptop, rclone, and an SDK
on another host. The flag is a protected action, and it is `false` for the app
in this recipe. A person sets it in the console; the platform answers a `403
protected_action` error to an agent. Your container's own access does not
depend on it.

## The files

Four files at the repo root.

### app.py

```python
import hmac
import logging
import os

import boto3
from botocore.exceptions import ClientError
from fastapi import Depends, FastAPI, Header, HTTPException, UploadFile
from fastapi.responses import Response

# uvicorn's logger, so these lines show up in the channel's logs.
logger = logging.getLogger("uvicorn.error")

BUCKET = os.environ["S3_BUCKET"]

# Your own secret, set with set_env. Reads stay public; writes carry it.
WRITE_TOKEN = os.environ.get("WRITE_TOKEN", "")

# The platform injects all five S3_* variables once the channel's blob store
# is ready. S3_ENDPOINT is this cell's own gateway on the private overlay
# network, so plain http and stock SigV4 are correct here — no botocore
# Config, no TLS options.
s3 = boto3.client(
    "s3",
    endpoint_url=os.environ["S3_ENDPOINT"],
    aws_access_key_id=os.environ["S3_ACCESS_KEY_ID"],
    aws_secret_access_key=os.environ["S3_SECRET_ACCESS_KEY"],
    region_name=os.environ["S3_REGION"],
)

app = FastAPI()

# Object names arrive from upload filenames and URL paths, so they are
# untrusted. The gateway rejects traversal too, but a 400 here is a clearer
# answer than its 403.
_BAD_SEGMENTS = frozenset({"", ".", ".."})


def _checked_key(key: str) -> str:
    if not key or any(segment in _BAD_SEGMENTS for segment in key.split("/")):
        raise HTTPException(status_code=400, detail="Invalid object name.")
    return key


def require_write_token(authorization: str = Header(default="")) -> None:
    # No token configured means no writes at all. Failing closed keeps a
    # forgotten WRITE_TOKEN from leaving the store open to anyone.
    if not WRITE_TOKEN:
        raise HTTPException(
            status_code=503, detail="Writes are not configured."
        )
    scheme, _, presented = authorization.partition(" ")
    # compare_digest, not ==: a wrong token must not be findable by timing.
    if scheme != "Bearer" or not hmac.compare_digest(
        presented.encode(), WRITE_TOKEN.encode()
    ):
        raise HTTPException(status_code=401, detail="Invalid write token.")


# boto3 blocks, so every route below is a sync `def` — FastAPI runs those in a
# threadpool instead of on the event loop.


# The health check probes GET / and needs a 2xx.
@app.get("/")
def root():
    return {"ok": True, "service": "recipe-blob-uploads", "bucket": BUCKET}


# Keys are plain and unprefixed: the gateway files every object under this
# channel's own private prefix, so "notes.txt" here can never collide with or
# reach another channel's objects. The one name to avoid is anything under
# ".xhost/", which is reserved for the platform.
@app.post("/files", dependencies=[Depends(require_write_token)])
def upload(file: UploadFile):
    key = _checked_key(file.filename or "")
    try:
        s3.upload_fileobj(file.file, BUCKET, key)
    except ClientError as exc:
        logger.error("upload of %s failed: %s", key, exc)
        raise HTTPException(
            status_code=502, detail="Could not store the file."
        ) from exc
    return {"key": key, "bytes": file.size}


@app.get("/files")
def list_files():
    try:
        # One page, up to 1000 objects. Use s3.get_paginator("list_objects_v2")
        # once you expect more than that.
        page = s3.list_objects_v2(Bucket=BUCKET)
    except ClientError as exc:
        logger.error("listing failed: %s", exc)
        raise HTTPException(
            status_code=502, detail="Could not list the files."
        ) from exc
    return {
        "files": [
            {
                "key": obj["Key"],
                "bytes": obj["Size"],
                "modified": obj["LastModified"].isoformat(),
            }
            for obj in page.get("Contents", [])
        ]
    }


@app.get("/files/{key:path}")
def download(key: str):
    checked = _checked_key(key)
    try:
        obj = s3.get_object(Bucket=BUCKET, Key=checked)
    except ClientError as exc:
        # A missing object can come back as AccessDenied instead of NoSuchKey:
        # the store will not confirm a key is absent to a caller scoped to one
        # prefix. Every key this app can name is inside its own channel's
        # prefix, so both codes mean the same thing here — no such file.
        if exc.response["Error"]["Code"] in ("NoSuchKey", "AccessDenied"):
            raise HTTPException(
                status_code=404, detail="No such file."
            ) from exc
        logger.error("download of %s failed: %s", checked, exc)
        raise HTTPException(
            status_code=502, detail="Could not read the file."
        ) from exc
    # Objects come back untyped: the gateway does not carry the upload's
    # Content-Type upstream. Keep your own metadata (a row in Postgres, a
    # naming convention) if a stored type matters to you.
    return Response(
        content=obj["Body"].read(), media_type="application/octet-stream"
    )


@app.delete("/files/{key:path}", dependencies=[Depends(require_write_token)])
def delete(key: str):
    checked = _checked_key(key)
    try:
        s3.delete_object(Bucket=BUCKET, Key=checked)
    except ClientError as exc:
        logger.error("delete of %s failed: %s", checked, exc)
        raise HTTPException(
            status_code=502, detail="Could not delete the file."
        ) from exc
    return {"deleted": checked}
```

Five details in that file are important.

**`BUCKET = os.environ["S3_BUCKET"]` runs at import.** This is a deliberate
trade-off, and it has a good result and a bad result. A channel whose blob
store is not `ready` has no `S3_*` variable in its environment. Then this line
raises `KeyError`, and the process does not start. The deploy fails on the
health check. That result is better than an app that answers `/` with a 200
and fails every `POST /files` with a 500. But the deploy still fails. If you
want a channel that starts without storage, read the variable in the handler,
and answer 503 when the variable is absent.

**Every route is a sync `def`, not `async def`.** boto3 blocks the thread. A
call that blocks in an `async def` handler stops the whole event loop. Then
one slow upload stops every other request in the process. FastAPI runs a plain
`def` handler in a threadpool, which is correct for an SDK that blocks.

**`_checked_key` rejects the name before the SDK sees it.** An object name
comes from the `filename` of an upload, or from the URL path. An attacker
controls both. The gateway also rejects a path traversal, so this check is a
second defence, not the only one. But a 400 with `Invalid object name.` is a
better answer than a 502 that hides the gateway's refusal.

**The `AccessDenied` branch in `download`.** This branch is unexpected. A
section below explains it.

**A download has the type `application/octet-stream`.** The gateway does not
send your upload's `Content-Type` upstream. Thus an object comes back with no
type, whatever the browser sent. If the stored type is important to you — for
example, you show the images in an `<img>` tag — keep the type yourself. Put
it in a Postgres row, or encode it in the key name. Then set the response type
from your own record.

### requirements.txt

```text
fastapi==0.115.6
uvicorn==0.34.0
python-multipart==0.0.20
boto3==1.43.59
botocore==1.43.59
```

FastAPI needs `python-multipart` to parse a `multipart/form-data` body.
Without it, an `UploadFile` parameter fails at request time, not at import
time. Thus the app starts, and then answers 500 on the first upload. This
recipe pins `boto3` and `botocore` to the same version on purpose. boto3
accepts only a narrow range of botocore versions. If you pin one of the two
only, the other one can change.

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

## The deploy

Every app owns a git repo. **`git push`, then `deploy`, is the standard
path.** A push sends only the diff. Thus each edit after the first one costs a
few lines, not the full contents of every file through a tool call. Use
`commit_files` in one situation only: git is not available on the machine that
you work on. **A push stores your code; it does not deploy the code.**
`deploy` is a separate call, so you name the commit that goes live.

There are five steps: the four steps that every app takes, and the write
token. If this is your first app, the
[static site recipe](https://docs.xhostd.com/guides/recipes-static) explains
the four common steps in more detail.

**1. Create the app.** This call provisions the `prod` channel, its hostname,
its git repo and its object store.

```text
create_app(name="recipe-blob", template="app")
→ {"id": "efb0f79a-ab98-4bb2-ad8a-40b27ab3f7fc",
   "name": "recipe-blob",
   "template": "app",
   "repo_url": "https://git.xhostd.com/docs/recipe-blob.git",
   "channels": [{"id": "21eb795a-7071-407b-8cf0-0e4f940af3c8",
                 "name": "prod",
                 "hostname": "recipe-blob-docs.xhostd.app",
                 "current_sha": null}], ...}
```

Keep the app `id` and the `prod` channel `id`. Every later call takes one or
both.

**2. Mint a credential.** The token is your git password, and it is valid for
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

**3. Clone, commit, push.** A new xhostd repo is empty, and git tells you so.
The warning is correct; it is not a fault:

```bash
$ git clone https://docs:$XHOST_TOKEN@git.xhostd.com/docs/recipe-blob.git
Cloning into 'recipe-blob'...
warning: You appear to have cloned an empty repository.
```

Write the four files into that directory. Then commit and push them:

```bash
$ cd recipe-blob
$ git add -A
$ git commit -m "blob-uploads recipe"
$ git push origin master
```

Do not put the token in a file that you commit, or in text that you paste.
`$XHOST_TOKEN` above holds the value from `get_credentials`. A remote URL with
a real token in it stays in `.git/config`, in your shell history and in the
output of every `git remote -v`.

**4. Set the write token.** The two write routes check this value.

```text
set_env(app_id="efb0f79a-ab98-4bb2-ad8a-40b27ab3f7fc",
        key="WRITE_TOKEN",
        value="<a long random value of your own — redacted here>",
        secret=True)
```

`secret=True` is important. MCP never gives a secret value back: `list_env`
returns the key, its kind, its scope and its change time, but a null value. To
read the value, use the click-to-reveal control in the web console, or the
HTTP API call `GET /apps/{app_id}/env/{key}/value`. The platform audits every
reveal. A plain variable is the default, and `list_env` returns it as
cleartext. Cleartext is the wrong place for a credential that permits writes
to your object store.

The order of step 4 and step 5 is important. The platform injects the
environment into a container when the container starts. Thus the token must
exist before the deploy that needs it. If you set the token first, the
container from step 5 already holds it. If you set the token after step 5, the
app refuses every write until you deploy again.

**5. Deploy the branch.**

```text
deploy(app_id="efb0f79a-ab98-4bb2-ad8a-40b27ab3f7fc",
       channel_id="21eb795a-7071-407b-8cf0-0e4f940af3c8",
       ref="master")
→ {"deploy_id": "f81e4372-9891-429d-bf43-ebc8d9259a10",
   "channel_id": "21eb795a-7071-407b-8cf0-0e4f940af3c8",
   "status": "queued"}
```

`ref` is a branch name. xhostd resolves it to the current head of that branch,
so after a push you do not need the sha. `deploy` also accepts `sha` when you
want an exact commit, and `sha` has priority if you pass both. The deploy then
runs in the background. Use `get_deploy_log` to follow it.

## Verify it

### The deploy log

This log comes from the deploy above,
`f81e4372-9891-429d-bf43-ebc8d9259a10`. It does not include the buildkit lines
that teach nothing. Every deploy of the channel writes the same set of lines.

```text
...
[2026-07-31T22:00:19+00:00] [build] #7 0.482 Resolved 20 packages in 313ms
[2026-07-31T22:00:21+00:00] [build] #7 1.738 Installed 20 packages in 73ms
[2026-07-31T22:00:21+00:00] [build] #7 1.738  + boto3==1.43.59
[2026-07-31T22:00:21+00:00] [build] #7 1.738  + botocore==1.43.59
[2026-07-31T22:00:21+00:00] [build] #7 1.738  + python-multipart==0.0.20
...
[2026-07-31T22:00:30+00:00] [build] image 1007.98 MB total, 59.33 MB charged — base xhost-runtime:node22-py313 exempt
[2026-07-31T22:00:32+00:00] channel snapshot saved: 0.00 MB
[2026-07-31T22:00:33+00:00] blob snapshot saved: ts=2026-07-31T22:00:32.447000+00:00
[2026-07-31T22:00:33+00:00] blob record pushed to cell gateway
[2026-07-31T22:00:33+00:00] health_check container=ac9d93ec09c193f560674fdd7a0dec2eb4adadce473cdc4921ed824dbd67a8ef port=3000 timeout=120.0s
[2026-07-31T22:00:33+00:00] [container] [xhost] starting launch.sh (XHOST_HTTP_PORT=3000) ...
[2026-07-31T22:00:36+00:00] [container] INFO:     Uvicorn running on http://0.0.0.0:3000 (Press CTRL+C to quit)
[2026-07-31T22:00:37+00:00] health_check ok
[2026-07-31T22:00:37+00:00] [container] INFO:     10.77.1.5:56020 - "GET / HTTP/1.1" 200 OK
[2026-07-31T22:00:37+00:00] caddy ensure_route hostname=recipe-blob-docs.xhostd.app upstream=10.77.1.5:32044
[2026-07-31T22:00:38+00:00] deploy success
```

Two of those lines are specific to a channel with a blob store.

**`blob snapshot saved: ts=...`.** Before your new container replaces the old
container, the platform marks a moment in the channel's object store. It does
the same for the channel's Postgres schema. The platform writes one marker
object, not a copy. The store keeps the full version history, so a timestamp
is enough to return every object to its state at that moment. You do not
request the marker, and you cannot omit it.

**`blob record pushed to cell gateway`.** This line explains why your key pair
works. The platform gives the channel and its scoped upstream credential to
the cell-local gateway. It does this before the container that uses them
starts. If a deploy reaches `health_check` without this line, check the S3
calls from the container first.

The line above them is `image 1007.98 MB total, 59.33 MB charged`. It is the
standard image-size report: only the layers that you add on the warm
`xhost-runtime` base count against your plan's cap.

### The curl

The transcript below shows the full sequence against the live channel. It
shows the public reads, the two conditions that refuse a write, and the two
writes that carry the token. `$WRITE_TOKEN` holds the value from step 4. The token itself
never appears in a transcript.

```console
$ curl -sS https://recipe-blob-docs.xhostd.app/
{"ok":true,"service":"recipe-blob-uploads","bucket":"recipe-blob-docs"}

$ printf 'the quick brown fox\n' > notes.txt

$ curl -sS -w ' [%{http_code}]\n' -F 'file=@notes.txt' \
    https://recipe-blob-docs.xhostd.app/files
{"detail":"Invalid write token."} [401]

$ curl -sS -w ' [%{http_code}]\n' -F 'file=@notes.txt' \
    -H 'Authorization: Bearer not-the-token' \
    https://recipe-blob-docs.xhostd.app/files
{"detail":"Invalid write token."} [401]

$ curl -sS -F 'file=@notes.txt' \
    -H "Authorization: Bearer $WRITE_TOKEN" \
    https://recipe-blob-docs.xhostd.app/files
{"key":"notes.txt","bytes":20}

$ curl -sS https://recipe-blob-docs.xhostd.app/files
{"files":[{"key":"notes.txt","bytes":20,"modified":"2026-07-31T22:05:40.166000+00:00"}]}

$ curl -sS https://recipe-blob-docs.xhostd.app/files/notes.txt
the quick brown fox

$ curl -sS -w ' [%{http_code}]\n' -X DELETE \
    https://recipe-blob-docs.xhostd.app/files/notes.txt
{"detail":"Invalid write token."} [401]

$ curl -sS -X DELETE -H "Authorization: Bearer $WRITE_TOKEN" \
    https://recipe-blob-docs.xhostd.app/files/notes.txt
{"deleted":"notes.txt"}

$ curl -sS https://recipe-blob-docs.xhostd.app/files
{"files":[]}

$ curl -sS -w ' [%{http_code}]\n' https://recipe-blob-docs.xhostd.app/files/nope.txt
{"detail":"No such file."} [404]

$ curl -sS -w ' [%{http_code}]\n' --path-as-is 'https://recipe-blob-docs.xhostd.app/files/../etc/passwd'
{"detail":"Invalid object name."} [400]
```

That hostname is the live demo, so the `GET` calls above reach a real app. The
`POST` and `DELETE` calls always answer 401 there, because the demo's write
token is not public. To see the successful writes, you must deploy your own
app and use your own token. The last list does not reproduce either. After the
capture of this transcript, an upload put `notes.txt` back, so the file that
the delete removed is present again.

Read four points in that transcript.

**The reads carry no credential, and the app refuses a write in two
conditions.** `GET /`, `GET /files` and `GET /files/{key}` answer every
caller. `POST /files` and `DELETE /files/{key}` answer 401 when the request
has no `Authorization` header. They answer 401 again when the header holds a
wrong token. The body is the same in both conditions, because a different body
tells an anonymous caller which part of the request was correct. `GET /` is
public for a second reason: the deploy's health check probes that route.

**The key in that list is `notes.txt`, with no prefix in front of it.** The
channel prefix exists upstream, and the gateway applies it on every call. The
prefix is never part of the name that your code handles.

**The app checks the last two calls itself; the store does not.** A key that
is absent gives a 404. `_checked_key` refuses a name that points outside the
prefix, and answers 400 before the app calls boto3.

**The transcript omits one path on purpose: the 503.** To capture that path,
the app must run without a `WRITE_TOKEN`. This channel is a public demo, and a
deploy without a write token puts the demo in the exact state that the token
prevents. Thus this guide describes the behaviour, and does not capture it.
Without a token in the container, `POST /files` and `DELETE /files/{key}`
answer `503` with `{"detail":"Writes are not configured."}`. They give that
answer to every caller, with a correct header or without one. The reads
continue to work. The list of failure modes below tells you what to do.

## Object versions, deletes and your usage number

Every channel's store keeps all versions of an object. This behaviour is S3
object versioning, and you cannot turn it off. Two results follow, and both
surprise people.

**A second write to a key does not replace the object.** A `PUT` to a key that
exists adds a new current version. It makes the previous version noncurrent.
The old bytes stay in the store.

**A delete does not erase anything either.** The store writes a delete marker,
and that marker becomes the new head of the key. The key is not in a list any
more, and a GET answers as if the object is absent. But the versions below the
marker stay in the store.

This history is deliberate, and it makes the `blob snapshot saved` line above
useful. A snapshot is a timestamp. A restore finds the version of each key
that was current at that moment, and writes that version again as the new
head. A restore only adds versions. Thus a restore cannot destroy another
snapshot, and you can undo a restore.

The usage number in `get_blob_usage` and on the channel stats page counts
**live bytes only**. Live bytes are the latest version of each key that has no
delete marker. A noncurrent version and a key with a delete marker do not
count against your plan's quota. Thus a delete does free space, although the
store erases nothing. The gateway does not compute the usage on the request
path, so the new number appears about one minute later. After a large delete,
wait about one minute before you decide that the quota is wrong.

## When it goes wrong

### A missing object comes back as `AccessDenied`, not `NoSuchKey`

This behaviour is unexpected, and it can cost you hours of work. The example
handles `NoSuchKey` and `AccessDenied` in the same branch. Both codes are
necessary there. If your code handles `NoSuchKey` only, a request for a key
that does not exist — `GET /files/nope.txt` — misses that branch. The request
then goes to the generic branch, and the app answers **502** instead of 404.

The cause is the isolation guarantee, and the guarantee is correct. Your
channel's upstream credential has a scope: your channel's own key prefix. The
storage provider does not confirm to such a caller that an object is *absent*.
The answer "no such key" is itself information about a name space that the
caller has no permission to see. Thus the provider answers `AccessDenied`. The
gateway sends the upstream status and body on without a change, so boto3
receives `AccessDenied`.

Your app has no real ambiguity. Every key that your code can name is inside
your own channel's prefix. Thus an `AccessDenied` on a GET has one meaning
only: there is no such object. Treat both codes as 404. The `download` route
does this:

```python
if exc.response["Error"]["Code"] in ("NoSuchKey", "AccessDenied"):
```

Note what that branch does *not* do: it raises the 404 and does not call
`logger.error`. A request for a file that is absent is an ordinary answer, not
a fault. Thus the correct behaviour writes nothing: a 404, and no line in the
runtime log. If you remove `AccessDenied` from that tuple, the same request
goes to the branch below. That branch writes the botocore error to the log and
answers 502. A 502 where you expect a 404 is the symptom to look for.
[Best practices](https://docs.xhostd.com/guides/bkm) explains how to read the
runtime log.

### An upload returns 507

A `507 Insufficient Storage` from the gateway means that the PUT goes past
your plan's blob quota. The quota is per user. It is the sum over every
channel of every app that you own. Thus one channel can cause a 507 on another
channel.

Delete the objects that you do not need. Then send the PUT again. The new
space appears about one minute later, because the gateway computes the usage
off the request path. Thus the first new PUT can fail, and a later one
succeeds.

Look at how your own code reports the 507. The single `except ClientError` in
this example changes a 507 into the app's generic 502 `Could not store the
file.` That answer is true, but it does not help the person who uploads the
file. To tell the user that there is no more space, test the status code:

```python
if exc.response["ResponseMetadata"]["HTTPStatusCode"] == 507:
    ...
```

### `KeyError: 'S3_BUCKET'` at startup

The channel's blob store was not `ready` at the time of the deploy. Thus the
platform injected no `S3_*` variable, and `os.environ["S3_BUCKET"]` at module
level raised the error. The container exits immediately. The health check
finds nothing on the port. The deploy then fails on a timeout, and the runtime
log holds the traceback.

The platform provisions a store in a very short time, and it sweeps again for
a store that stays in `provisioning`. Thus the usual repair is a new deploy.
If you prefer an app that starts without storage, read the variables in the
handlers, not at module scope.

### Every write answers 503 `Writes are not configured.`

The container has no `WRITE_TOKEN`, or it has an empty one. The check fails
closed, so the app refuses every upload and every delete. Without a token to
compare, the only safe answer is a refusal. The reads still work. Thus the
deploy passed, and the first upload showed you the fault.

Set the value with
`set_env(app_id, key="WRITE_TOKEN", value=..., secret=True)`. Then deploy the
channel again. A change to the environment reaches the container at the next
deploy only. Thus a change against a live channel does nothing until you
deploy that channel again.

### A write answers 401 `Invalid write token.`

The app has a write token, and the request does not match it. Three conditions
give this answer. First, the request has no `Authorization` header. Second,
the header scheme is not `Bearer`; a common error is a bare token as
`Authorization: <token>`. Third, the token does not match the configured one.
The answer is the same for all three on purpose, because a different answer
tells an anonymous caller which part of the request was correct.

The 503 above and this 401 are different codes on purpose. The 503 means that
the app has no write token. The 401 means that *your* request has no correct
credential. If you are sure that you sent the correct value, confirm the value
itself. `list_env` never returns a secret. Thus you cannot recover a lost
token; you set a new one with another `set_env`.

### You added a botocore `Config`

The symptoms are signature errors: `SignatureDoesNotMatch`, or a 400 from a
request that looks correct. They occur on a client that worked before your
edit, or on the first call of a client that you copied from an AWS example.
The gateway uses stock SigV4 on the injected endpoint, and boto3's defaults
are correct for it. Delete the `Config`. Pass only `endpoint_url`, the key
pair and `region_name`.

### You want an external-access toggle

`external_blob_access_enabled` is not the control that makes your app's
storage work. A change to it repairs no fault inside the container. It
controls whether the public S3 gateway gives your channel's credentials to a
client *outside* the platform. It is a protected action: a person sets it in
the console, and the platform answers a `403 protected_action` error to an
agent. It changes nothing on the injected `S3_ENDPOINT` path that your
container uses. If the S3 calls in your app fail, the cause is elsewhere in
this list.

### Files come back with the wrong type

The store keeps an object with no type. The gateway does not send the upload's
`Content-Type` upstream. Thus a GET has no type to return, and the example
answers `application/octet-stream`. A browser that receives that type
downloads the file; it does not show it. Keep the type yourself: a column next
to the key in Postgres, or a file extension that you map. Then set the
`media_type` of the response from your own record.

### You wrote a key under `.xhost/`

The platform keeps that sub-prefix for its deploy snapshot markers. The
gateway refuses any key under it with a 403, on a write and on a read. The
gateway also removes its own markers from your lists. Thus you never see the
platform's objects, and you cannot put your own objects beside them. boto3
raises the refusal as a `ClientError`, and this example answers it with the
generic 502. Use another name.
