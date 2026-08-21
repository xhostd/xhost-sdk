# Recipe: a raw TCP service on a public port

## What you get

A Python service on the `app` template. It speaks a line protocol over **raw
TCP** on a public `host:port`, and it serves no HTTP.

The platform moves the bytes between the public address and one fixed port in
your container. It does not examine them: it terminates no TLS, it knows no
protocol, and it parses nothing. The endpoint is therefore protocol-agnostic. A
game server, a message broker, gRPC, a database wire protocol and an `sshd` that
you install are all the same to the platform.

The app is `recipe-tcp`, the template is `app`, and the channel is `prod`.
`expose_port` allocates the public raw-TCP address in
[The deploy](#the-deploy) below. The address does not change while the exposure
exists, so you can give it to your users.

Two conditions control this recipe before your code is important. First, your
**plan** must include port forwarding; the basic plan does not. Second, a person
must set the project's **port forwarding toggle** to on in the web console. The
sections below give the full detail on both. There is no tool for the toggle,
and the platform refuses it to an agent, so an agent must ask the user.

**"Can I SSH into my app?"** Yes, and this recipe is the method. xhostd runs no
SSH daemon and no SSH gateway, and it will not add one. To route SSH for many
tenants through one port, a gateway must join the key exchange. That gateway
must also translate an xhostd token into SSH authentication, and it must manage
the host keys. A route on the destination port needs none of that.

So you install your own `sshd` in your container, and you bind it to
`$XHOST_FORWARD_PORT`. Then you connect to your endpoint with your own keys,
your own config and your own users. `scp`, `sftp` and `-L` all work, because the
daemon is a real `sshd`.

The costs are these. The method works only on the `app` and the `docker`
templates, never on `static`. It uses the one forward port of the channel. xhostd
does not install, supervise or support your `sshd`.

## The files

Two files at the repo root. The service uses only the standard library, so it
needs no `requirements.txt`. `install.sh` is optional, and this recipe has
none.

### server.py

```python
"""A raw-TCP service on the channel's public forward address.

The protocol is a toy (PING / ECHO / TIME / QUIT); the shape around it is the
part to copy. This channel serves no HTTP at all, so the deploy's HTTP probe
can never pass — readiness is signalled by creating $XHOST_READY_FILE once the
listener is up, which is the other signal the health check accepts.
"""

import logging
import os
import socketserver
import sys
from datetime import UTC, datetime
from pathlib import Path

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger("tcp-forward-service")

# The platform injects this into every app container. Read it from the
# environment, so that the app still works if the platform changes the value.
PORT = int(os.environ["XHOST_FORWARD_PORT"])

# The endpoint is public and unauthenticated, so a line is bounded: a client
# that sends bytes and never a newline must not grow a buffer without limit.
MAX_LINE = 4096


class LineHandler(socketserver.StreamRequestHandler):
    def handle(self):
        # Connections are relayed byte-for-byte, so this address is the
        # forwarding node's, never the real client's.
        logger.info("connection from %s", self.client_address[0])
        while True:
            raw = self.rfile.readline(MAX_LINE)
            if not raw:  # the client hung up
                return
            verb, _, rest = raw.decode("utf-8", "replace").strip().partition(" ")
            match verb.upper():
                case "PING":
                    reply = "PONG"
                case "ECHO":
                    reply = rest
                case "TIME":
                    reply = datetime.now(UTC).isoformat()
                case "QUIT":
                    self.send_line("BYE")
                    return
                case other:
                    reply = f"ERR unknown command {other!r}"
            self.send_line(reply)

    def send_line(self, text):
        self.wfile.write(f"{text}\n".encode())


class LineServer(socketserver.ThreadingTCPServer):
    # One thread per client, so a slow client never blocks the others.
    daemon_threads = True
    # A restarted container would otherwise fail to bind while the previous
    # sockets sit in TIME_WAIT.
    allow_reuse_address = True

    def handle_error(self, request, client_address):
        # The default prints a full traceback for what is usually just a client
        # that vanished mid-line. Log one line; the listener carries on.
        logger.warning("client %s dropped: %s", client_address[0], sys.exception())


def main():
    with LineServer(("0.0.0.0", PORT), LineHandler) as server:
        # Create the ready file HERE, not before. The socket is bound and it
        # accepts connections when this object exists, so this is the first
        # correct moment to report "ready". A launch.sh that creates the file
        # at the top marks the channel healthy while it refuses every client.
        Path(os.environ["XHOST_READY_FILE"]).touch()
        logger.info("listening on 0.0.0.0:%s", PORT)
        server.serve_forever()


if __name__ == "__main__":
    main()
```

Four parts of that file are important.

**The port comes from `$XHOST_FORWARD_PORT`.** The platform publishes a second
port on every non-`static` container, next to the HTTP health port. The
container-side number is one platform constant, `7000`. The platform sets
`XHOST_FORWARD_PORT` in every container, also when the channel has no exposure.
A hardcoded `7000` gives the same result. Read the variable anyway: it costs one
line, the variable is the documented contract, and the number is an
implementation detail. It is also the same rule as `$XHOST_HTTP_PORT`. The key
is reserved: `set_env` refuses your own value for it. The platform does not
accept your value and then replace it.

**The app creates the ready file after the socket listens, not before.** This is
the key rule of the recipe, and [Verify it](#verify-it) shows it on a live
deploy. The deploy's health check accepts one of two signals, whichever comes
first. The first signal is an HTTP 2xx/3xx from `GET /` on the health port. The
**other** signal is the file with the name in `$XHOST_READY_FILE`.

A TCP-only app serves no HTTP, so it can never satisfy the first signal. There
is no template switch that stops the HTTP probe. The ready file is therefore the
only way for a TCP-only channel to pass its deploy. A channel that creates the
file at the top of `launch.sh` passes its deploy, but it serves nothing: it
refuses every connection.

**Bind `0.0.0.0`, not `localhost`.** The relay connects to the container from
outside it, exactly as the HTTP probe does. It cannot reach a socket on
`127.0.0.1`.

**The server limits the line, and the handler writes a log line instead of a
traceback.** The endpoint is public, and xhostd authenticates nothing on a
forward port. Your app is therefore the only control on the connection.
`readline(MAX_LINE)` limits the buffer when a client sends bytes and no newline.
`handle_error` writes one log line for a client that stops in the middle of a
line, and not one traceback for each lost peer.

Look at the `client_address` in the handler's log line. The connections come
from the forward node, not from the real client, so that address is the address
of the relay. If you need the true peer address, put it in your own protocol.

### launch.sh

```sh
#!/bin/sh
# Runs at BOOT, as the non-root 'app' user. Never install anything here.
# There is no install.sh: this service uses only the standard library.
set -eu

exec python3 server.py
```

`exec` makes the Python process PID 1. The process then gets the stop signals
directly, and not through a shell that discards them. This is more important
here than in an HTTP recipe. A container that uses the full stop timeout keeps
every relayed session open for that time.

## The deploy

Five steps. The first one is not a tool call.

### Step 0 — turn the project toggle on, in the console

`create_app` reports two different fields, and each one tells you a different
thing:

- **`port_forwarding_available`** — the account's **plan** permits port
  forwarding. It is false on the basic plan.
- **`port_forwarding_enabled`** — the **project toggle**. Its default is
  **false**, also when the plan permits port forwarding.

The toggle is a **protected action**. A project admin sets it under
**Port forwarding** on the settings page for the project. An agent that calls
the toggle route gets a `403 protected_action` error, and that message also
names the settings page. A user who wants an agent to set the toggle turns
**agent access** on for the project first, in the Agent access card of the same
page. You do not have to build the URL for `expose_port`, because the error for
a toggle that is off contains it:

```text
expose_port(app_name="recipe-tcp", channel="prod")
→ permission_denied: Public TCP endpoints are turned off for this project.
  A project admin must turn them on first at
  https://console.xhostd.com/projects/cb2ecd27-a8dd-4882-99de-fb191f0c8a00/settings
```

The id in the URL is the app `id` from `create_app`. There is no MCP tool for
the toggle, by design. A person must make the decision to open a project to the
public internet. An agent with the power to set the toggle could also report a
live endpoint that the user did not agree to. If you are an agent, **ask the
user to set the toggle** and wait. The platform answers a `403
protected_action` error to you, until the user turns agent access on.

The toggle is ADMIN-only, and not MEMBER-only, because it controls
`expose_port`. A member must not re-open every endpoint that an admin closed.

All the steps below work with the toggle off, up to `expose_port`.
`expose_port` fails with a 403. Set the toggle first.

### Step 1 — create the app

```text
create_app(name="recipe-tcp", template="app")
→ {"id": "cb2ecd27-a8dd-4882-99de-fb191f0c8a00",
   "name": "recipe-tcp",
   "template": "app",
   "repo_url": "https://git.xhostd.com/docs/recipe-tcp.git",
   "owner_username": "docs",
   "port_forwarding_enabled": false,
   "port_forwarding_available": true,
   "channels": [{"id": "a9832287-7684-4f78-94dc-9795761c60d8",
                 "name": "prod",
                 "hostname": "recipe-tcp-docs.xhostd.com",
                 "current_sha": null,
                 "status": "provisioning"}],
   ...}
```

`port_forwarding_available: true` with `port_forwarding_enabled: false` is the
normal state of a new app on a paid plan. The plan gives the capability, but
nobody set the project toggle. Keep the app id and the `prod` channel id,
because the deploy calls take both.

### Step 2 — push the files

Every app owns a git repo, and **`git push` → `deploy` is the standard path**.
Use `commit_files` only when the machine has no git. The rest of the recipe is
the same for both methods. A deploy reads a commit from the repo, and it does
not know the source of that commit.

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
$ git clone https://docs:$XHOST_TOKEN@git.xhostd.com/docs/recipe-tcp.git
Cloning into 'recipe-tcp'...
warning: You appear to have cloned an empty repository.
```

Write `server.py` and `launch.sh` into that directory. Then commit the two
files and push them:

```bash
$ cd recipe-tcp
$ git add -A
$ git commit -m "tcp line service on the forward port"
[master (root-commit) 75087f0] tcp line service on the forward port
 2 files changed, 87 insertions(+)
 create mode 100755 launch.sh
 create mode 100644 server.py
$ git push origin master
To https://git.xhostd.com/docs/recipe-tcp.git
 * [new branch]      master -> master
```

Do not commit or paste the token. `$XHOST_TOKEN` above is the value from
`get_credentials`. A remote URL with a real token in it stays in `.git/config`,
in your shell history, and in the output of every `git remote -v`.

### Step 3 — deploy the branch

```text
deploy(app_id="cb2ecd27-a8dd-4882-99de-fb191f0c8a00",
       channel_id="a9832287-7684-4f78-94dc-9795761c60d8",
       ref="master")
→ {"deploy_id": "06ce028b-2157-477d-9d08-385abbc92939",
   "channel_id": "a9832287-7684-4f78-94dc-9795761c60d8",
   "status": "queued"}
```

`ref` is a branch name. xhostd resolves it to the current head of that branch,
so after a push you do not need the sha.

### Step 4 — expose the port

`expose_port(app_name="recipe-tcp", channel="prod")` allocates the public
endpoint of the channel. The port-forwarding tools take **names**, not ids,
unlike `deploy`. They take `app_name` and `channel`. Use
`owner_username/app_name` when two apps that you can access have the same name.

The optional third argument is `allow_cidrs`. It is a source-address allowlist
of 16 IPv4/IPv6 addresses or CIDR ranges at most (`203.0.113.7`,
`203.0.113.0/24`, `2001:db8::/32`). **An empty list, or no list, lets the whole
internet connect**, and that is the default. The allowlist is an additional
control, not authentication. A later call without the argument deletes an
allowlist that exists, as [The allowlist](#the-allowlist) below shows.

The call returns `{channel_id, channel, host, port, allow_cidrs, active,
created_at}`. Read `active` before you report success. `active` does not mean
"the row exists". It is true only when the project toggle is on and the owner's
plan includes port forwarding. `active: false` means that you have an address
that carries no traffic.

```text
expose_port(app_name="recipe-tcp", channel="prod")
→ {"channel_id": "a9832287-7684-4f78-94dc-9795761c60d8",
   "channel": "prod",
   "host": "fwd-1.xhostd.com",
   "port": 20582,
   "allow_cidrs": [],
   "active": true,
   "created_at": "2026-08-01T09:05:05.190500Z"}
```

`allow_cidrs: []` is the open case: the whole internet can connect. `active:
true` confirms that the two conditions are satisfied.

That host and that port come from the real run, so the response is the true
output. **Do not connect to that address.** xhostd released the endpoint, and a
released port goes back to the pool for another service. A connection to a port
from a transcript reaches a different service, not this recipe. xhostd allocates
a port for each exposure, so use the values from your own call.

Read these three properties of the address before you give it to your users:

- **It needs no redeploy.** The container already publishes its forward port,
  so `expose_port` only writes a row. The next connection carries traffic.
- **It does not change.** A redeploy keeps the same address. A second
  `expose_port` call on a channel that has an endpoint returns the *same* host,
  port and `created_at`. Only `allow_cidrs` takes the value of the new call,
  also when the new call gives no value. A release and a new exposure are
  different: they allocate a **new** address.
- **There is one endpoint for each channel.** A channel has one endpoint at
  most. `staging` and `prod` get different addresses. You cannot get a second
  port on `prod`.

## Verify it

### The deploy log

These six lines are the proof of the recipe. They are the end of deploy
`06ce028b-2157-477d-9d08-385abbc92939`. The `[...]` marks show where the
transcript omits other deploy lines: the build and backup lines above, and the
route and channel lines below. The transcript also shortens the container id:

```text
[...]
[2026-08-01T08:40:08+00:00] start_static_site template=app
[2026-08-01T08:40:08+00:00] health_check container=423fc265... port=3000 timeout=120.0s
[2026-08-01T08:40:08+00:00] [container] [xhost] starting launch.sh (XHOST_HTTP_PORT=3000) ...
[2026-08-01T08:40:09+00:00] health_check ok
[2026-08-01T08:40:09+00:00] [container] 2026-08-01 08:40:08,591 INFO listening on 0.0.0.0:7000
[...]
[2026-08-01T08:40:09+00:00] deploy success
```

Compare that log with the log of another recipe. Three points are different.

**The probe uses HTTP — `health_check ... port=3000 timeout=120.0s` — but
this app serves no HTTP.** The platform publishes the HTTP health port and
probes it for every template. It does this also when your app gives no answer
there. No process in this container binds port 3000. The line
`"GET / HTTP/1.1" 200` in the log of every other recipe is the HTTP arm of the
health check. That line cannot appear here, and the 120-second timeout can only
expire.

**`health_check ok` comes one second later.** That is the ready-file arm, and it
is the only arm that this app can satisfy. `server.py` created
`$XHOST_READY_FILE` immediately after it bound the socket. The probe accepted
the file and stopped, because the HTTP answer never comes. A deploy that reports
`port=3000`, and then succeeds one second later with no process on port 3000, is
not a contradiction. The ready file gave the signal.

**`listening on 0.0.0.0:7000`.** This is your own log line. It shows the value
that the platform set in `XHOST_FORWARD_PORT`.

One more detail: the build reported `image 948.68 MB total, 0.02 MB charged —
base xhost-runtime:node22-py313 exempt`. The build installed nothing on the warm
base image. A service that uses only the standard library therefore has an
almost zero build cost.

The channel also gets its HTTPS hostname, `recipe-tcp-docs.xhostd.com`. That
hostname returns **502**, and this is correct, because no process listens on the
health port. See [When it goes wrong](#when-it-goes-wrong).

### The endpoint is live

`list_exposed_ports(app_name="recipe-tcp")` returns every exposed channel of
the project in one call. The result is `{forwards: [{channel_id, channel, host,
port, allow_cidrs, active, created_at}]}`, in channel-name order. There is no
variant for one channel. Call it before you propose an `expose_port`: if an
entry exists, you only report the address. Call it also when a user reports a
broken endpoint. `active: false` tells you that the project toggle went off, or
that the owner's plan changed. That fault is very different from a service that
does not start.

```text
list_exposed_ports(app_name="recipe-tcp")
→ {"forwards": [{"channel_id": "a9832287-7684-4f78-94dc-9795761c60d8",
                 "channel": "prod",
                 "host": "fwd-1.xhostd.com",
                 "port": 20582,
                 "allow_cidrs": [],
                 "active": true,
                 "created_at": "2026-08-01T09:05:05.190500Z"}]}
```

### The round trip

The proof is a raw TCP exchange with the allocated address: no curl, no HTTP, no
TLS. In the code below, use the `host` and the `port` that `expose_port`
returned for **your** channel. The values in the transcript above belong to a
released endpoint. Any TCP client is sufficient: `nc <host> <port>` works, and
you can type the verbs by hand. The capture below uses Python, because Python
behaves the same on every machine, and the `nc` command does not:

```python
import socket

# The host and port expose_port returned for your channel.
HOST = "<host>"
PORT = <port>

s = socket.create_connection((HOST, PORT), timeout=15)
s.sendall(b"PING\nECHO hello there\nTIME\nBOGUS\nQUIT\n")
s.shutdown(socket.SHUT_WR)
while chunk := s.recv(4096):
    print(chunk.decode(), end="")
```

```text
PONG
hello there
2026-08-01T09:07:31.736818+00:00
ERR unknown command 'BOGUS'
BYE
```

Five requests and five replies, in order, over a plain TCP socket to a public
address. No component terminated TLS, and no component parsed a header. This
recipe invented the protocol, and the platform does not know it. That is the
purpose of a raw TCP port.

### The log in the container

`get_runtime_log` reads the log of the live container. With no `command`, it
reports what you can read, and it reads nothing:

```text
get_runtime_log(app_id="cb2ecd27-a8dd-4882-99de-fb191f0c8a00", channel="prod")
→ container #1 (xhost-cb2ecd27-a9832287-00000001) — running
  started: 2026-08-01T08:40:08.340361104Z
  readable containers: #1 (pass container_index to read an older one)
  no command given — pass one (e.g. "tail -n 200 app.log") to read the log itself
```

Give it a command, and it returns the log:

```text
get_runtime_log(..., command="tail -n 20 app.log")
→ log file: /log/app.log (257 bytes)
  2026-08-01T08:40:08.546775293Z [xhost] starting launch.sh (XHOST_HTTP_PORT=3000) ...
  2026-08-01T08:40:08.592035842Z 2026-08-01 08:40:08,591 INFO listening on 0.0.0.0:7000
  2026-08-01T09:07:31.736927506Z 2026-08-01 09:07:31,736 INFO connection from 10.77.0.1
```

Three lines for the full life of the app: the boot, the bound socket, and one
connection. That connection comes from [The round trip](#the-round-trip), and
its `TIME` reply has the same second, `09:07:31`.

**`10.77.0.1` is not the client.** It is the forward node, on the platform's own
network. The client that opened the socket was on the public internet. This log
line shows the fact in the `server.py` comment: the relay copies the bytes, so
your handler always sees the address of the relay. If you need the true source
address, put it in your own protocol.

### The allowlist

A second `expose_port` call on a channel that has an endpoint changes that
endpoint in place. This call set the list to `203.0.113.0/24`. That range is
TEST-NET-3, a documentation range, so the list excludes every real client, and
also the client of the round trip above:

```text
expose_port(app_name="recipe-tcp", channel="prod",
            allow_cidrs=["203.0.113.0/24"])
→ {"host": "fwd-1.xhostd.com",
   "port": 20582,
   "allow_cidrs": ["203.0.113.0/24"],
   "active": true,
   "created_at": "2026-08-01T09:05:05.190500Z"}
```

The host, the port and the `created_at` are the same as in the first exposure.
Only the allowlist changed.

**The platform does not refuse a blocked source.** This client connects from an
address outside the list, and asks for a `PING`:

```text
connect ok after 0.086s
recv after 0.197s: b''
```

The TCP handshake **succeeds**. Then the peer closes the socket with zero bytes.
There is no `ECONNREFUSED`, no error message, and no reply. A client that only
tests `connect()` reports the endpoint as healthy.

The app saw nothing of this:

```text
get_runtime_log(..., command="grep -c 'connection from' app.log")
→ log file: /log/app.log (257 bytes)
  1
```

The count is still one, from [The round trip](#the-round-trip). The blocked
connections did not reach the container. The platform applies the list above
your app: it accepts the socket, it compares the source address with the list,
and it closes the connection. No component connects to your container. A silent
log is therefore not proof of a fault in your app.

**A second call without `allow_cidrs` clears the list.** The argument does not
mean "keep the current list". It is the new value, and its default is the empty
list:

```text
expose_port(app_name="recipe-tcp", channel="prod")
→ {..., "allow_cidrs": [], ...}
```

The purpose of that call was to change nothing. The call opened the endpoint to
the whole internet again. The blocked client then completed a full round trip at
`09:13:36`. The platform gave no warning, and the response looks like every
other success.

Therefore, **give `allow_cidrs` in every `expose_port` call for a channel that
has a list**. Do this also for a call that you think changes nothing. If you do
not know the current list, read it with `list_exposed_ports` first. Then send
the same list back.

## When it goes wrong

### The deploy fails its health check and the app is fine

This deploy did not fail, so the message below is an example, with a short
container id. It names the two signals that the probe accepts:

```text
health check failed for container ...: no 2xx/3xx response at
GET / on port 3000 and no readiness file created at $XHOST_READY_FILE
within 120s
```

For a TCP-only app the first half is always true: the app serves no HTTP, and
you cannot stop the HTTP probe. The message therefore tells you one fact: your
app did not create `$XHOST_READY_FILE`. Create the file immediately after you
bind the socket, as `server.py` does, and not before.

### The ready file exists and the deploy still fails

The ready file is necessary, but it is not sufficient. The probe tests that the
container is alive **before** it tests the ready file, by design. A process can
create the file and then stop. A file from a dead process must not certify that
process. This recipe did not cause this failure either. The message below is an
example, with a short container name, and it comes immediately, not after 120
seconds:

```text
container xhost-... exited during boot (exit code 1)
```

That message tells you to read the runtime log. The health-check advice above
does not apply.

### 402 `plan_limit_exceeded` on `expose_port`

Your plan does not include public TCP endpoints. This error asks you to upgrade
the plan. A second call always gives the same error.

| Plan | Port forwarding | Concurrent connections per endpoint | Session time cap |
|---|---|---|---|
| basic | no | — | — |
| builder | yes | 10 | none |
| indie | yes | 10 | none |
| pro | yes | 10 | none |

The cap applies to each endpoint, and it is small, by design. One forward port
into one container is not more useful with more concurrent connections. A
downgrade deletes nothing: the rows stay, and `active` becomes false. The
platform then refuses new connections until you restore the plan. The same
address then works again, with no new allocation. The condition is the
**owner's** plan, so a shared project follows the owner, not the caller.

### 403 `permission_denied` on `expose_port`

The project's port-forwarding toggle is off. The error gives the console URL for
the toggle. [Step 0](#step-0-turn-the-project-toggle-on-in-the-console) quotes
the full error, and it explains why an agent cannot set the toggle. Ask the user
and wait. Then send the same call again.

Do not confuse this error with the 402 above. A 402 means "your plan does not
permit this". A 403 means "your plan permits this, but the project toggle is
off".

### 400 on a `static` app

A static site is a standard nginx with a read-only content mount. You supply
files, not a process, so no process in the container can bind the forward port.
The error says this, and it names the `app` and the `docker` templates. The
platform sets the template of a channel when it creates the container, and a
redeploy does not change it.

### 409 on `expose_port`

`no public port is available right now`. The forward fleet has no free port.
This error is temporary, unlike a 402 or a 403, so send the call again after a
short wait. It is a capacity signal for the operator, and you cannot correct it
from the app.

### The connection is refused, or opens and closes at once

The two symptoms have different causes:

- **The socket opens, then closes at once with no bytes.** This is the common
  case, and it is easy to read incorrectly. The output is `connect ok after
  0.086s` and then `recv ... b''`. The measurements show this result for [a
  blocked source](#the-allowlist) and for a released endpoint. A component
  *above* your app refused the connection. The cause is one of these:

    - The platform released the endpoint.
    - Your source IP is not in `allow_cidrs`.
    - The project toggle is off.
    - The plan does not permit port forwarding.
    - The channel does not run.
    - The endpoint is at its concurrent-connection cap.

  Your app's log shows nothing, because nothing reached the app. Read `active`
  in `list_exposed_ports` to separate the policy causes from the others.
- **Connection refused.** A released endpoint and a blocked source do *not* give
  this result. The node accepts the connection first, and applies the policy
  after, so a policy denial always gives the case above. A true refusal means
  that no process listens on that port on the node. The usual causes are a wrong
  port, or a node that is down.
- **The node ignores repeated attempts.** The node refuses a source IP several
  times in one minute, then it puts that IP in a short cooldown. It then drops
  the connections from that IP with no round trip. Wait for the end of the
  cooldown, and do not send more attempts.

### The HTTPS hostname returns 502

This is correct, and it is not a fault:

```bash
$ curl -sS -o /dev/null -w '%{http_code}\n' https://recipe-tcp-docs.xhostd.com/
502
```

The channel gets `recipe-tcp-docs.xhostd.com`, and Caddy routes that hostname.
No process listens on the health port, so the proxy has no upstream. A 404 from
the hostname is different: it tells you that the channel has no route, because
the deploy did not reach `caddy ensure_route`.

To get both, serve HTTP on `$XHOST_HTTP_PORT` *and* TCP on
`$XHOST_FORWARD_PORT` from the same container. The platform permits this, and
the HTTP arm of the health check then passes.

### You unexpose the port and the traffic continues

`unexpose_port` releases the address, and it reports this clearly:

```text
unexpose_port(app_name="recipe-tcp", channel="prod")
→ {"result": "public endpoint released for channel prod"}
```

The platform refuses new connections from that moment. A connection immediately
after the call gave the same result as a blocked source: `connect ok after
0.140s` and then `recv ... b''`. **The platform keeps the sessions that
exist.** It examines the authorization once, when it accepts the connection, and
it does not examine it again. After that, the relay moves the bytes and resolves
nothing.

This is a measurement, not a deduction. One client held one connection open for
the full sequence, and it sent an `ECHO` tick every five seconds. Every tick
after the release came back correctly. The ticks 20 to 26 all come after the
release: 31 more seconds of traffic on a released endpoint. The client opened
that session before the call.

To also stop the open sessions, do two steps in this order. **The order is
important**:

1. `unexpose_port` removes the address from the allocator, so no client can
   connect to it again.
2. `deploy` the channel. The new container replaces the old one, and this closes
   the upstream connection of every relay. The data path stops at once.

The deploy stopped the held session, and the logs agree to the second. From
deploy `2dd5f25e-bf6d-4d99-a1ea-9708e317b727`:

```text
[2026-08-01T09:16:01+00:00] start_static_site ok: container=627ae2a5c5dd...
[2026-08-01T09:16:01+00:00] [container] listening on 0.0.0.0:7000
[2026-08-01T09:16:02+00:00] stop_and_remove old container=423fc265...
[2026-08-01T09:16:07+00:00] stop_and_remove ok
```

The log of that client, in UTC+3:

```text
12:16:02 tick 26: tick-26
12:16:07 tick 27: EOF — session dropped
```

`12:16:07` local is `09:16:07` UTC. That is the same second when
`stop_and_remove` completed, five seconds after its start. The session continued
through the start of the new container, and it stopped with the old container.
That is the rule: a session belongs to the container that accepted it.

Those five seconds are the stop timeout, so a process that ignores SIGTERM keeps
its connections for that time. A deploy that fails before the cutover stops no
session. Confirm that the deploy reports `success`, and not only that the
platform queued it.

A deploy before `unexpose_port` gives no result. That deploy keeps the endpoint
allocated, and it points the endpoint at the new container, so the peer connects
again to the replacement. With `unexpose_port` first, the peer has no address
for a new connection.

A new exposure after that gives a **new** address, not the old one. The same
channel came back on port `28198`, and its first port was `20582`. Its
`created_at` is `2026-08-01T09:19:55.119029Z`, against the first
`2026-08-01T09:05:05.190500Z`. The new timestamp shows that the platform made a
new endpoint, and did not activate the old one again. Tell every client that has
the old address.

### A connection dropped for no reason

A forwarded connection can stop at any time, so your client must connect again.
There is no drain, and the platform does not move a connection to another
process. A platform deploy restarts the forward process, and every session on
that process stops at that moment. TCP keepalive also removes a dead peer after
approximately two minutes. A lost client therefore does not keep a slot against
your concurrent-connection cap. There is no idle timeout, by design, because a
quiet connection is normal on a port with no protocol.

## See also

- [Best known methods](https://docs.xhostd.com/guides/bkm) — the health-check
  checklist, plan budgets, and how to read a quota error.
- [Recipe: Python API on the app template](https://docs.xhostd.com/guides/recipes-app-python)
  — the same template with HTTP, `install.sh` and dependencies.
- [Docker recipe: your own Dockerfile](https://docs.xhostd.com/guides/recipes-docker)
  — for a service that needs its own base image. The platform publishes the
  forward port for the `docker` template too, on exactly the same conditions.
