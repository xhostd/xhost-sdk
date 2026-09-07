# Register as an agent

This guide is for an agent that works with no person at a browser. You open
an xhostd account yourself, with an SSH key and no Google sign-in. The
platform answers a token, and you hold every tool and every route with it.

You need a machine that runs commands. A runtime with no shell, such as the
claude.ai connector, cannot sign a message, and a person connects it through
OAuth instead. Read
[Getting started](https://docs.xhostd.com/getting-started) for that path.

What you get:

- An account on the `starter` plan. The caps are the `starter` row of
  `GET /plans`, and the registration response repeats them:

  | Field | Value | Meaning |
  |---|---|---|
  | `max_channels` | 1 | One app, with its `prod` channel |
  | `cpu_soft_cores` | 0.1 | The fair share under contention; bursts to 2x |
  | `mem_limit_mb` | 128 | Memory per container |
  | `blob_storage_bytes` | 134217728 | 128 MiB of object storage, enforced |
  | `storage_mb` | 250 | Database storage, advisory |
  | `image_size_bytes` | 536870912 | 512 MiB charged image size |
  | `port_forwarding` | false | No public raw-TCP endpoint |

  The [pricing page](https://xhostd.com/pricing) lists `starter` beside the
  other tiers.
- A private key at `~/.ssh/xhost_ed25519`. The key is the durable credential.
  It never leaves the machine, and it renews the token without a person.
- A 30-day `xh_` token with the default scopes. The token is your git password
  over HTTPS, your Postgres password, and your bearer for the API and the MCP
  server. When it expires, the key mints a new one.

## Before you start

You need three tools on the machine:

- `ssh-keygen` from OpenSSH 8.0 or later. Older versions have no `-Y sign`.
- `curl`.
- `python3`, for the two short scripts that build the request body and
  store the token.

The machine's clock must sit within 300 seconds of the platform clock. The
signed message carries a Unix time, and the platform refuses one outside that
window.

Run every command in a subprocess. No secret then enters a tool call or the
transcript.

## Step 1: the key

Use exactly this path. It is the same key `register_ssh_key` uses, so one key
serves the registration and every `git push`.

```sh
mkdir -p ~/.ssh; [ -f ~/.ssh/xhost_ed25519 ] || ssh-keygen -t ed25519 -N "" -f ~/.ssh/xhost_ed25519
```

The command makes a key only when the file is absent. An existing key on this
machine is either registered already, in which case you sign in with it
(read [Renew the token](#renew-the-token)), or it registers now.

The platform accepts `ssh-ed25519` alone on this path. A person can paste an
RSA or ECDSA key in the console for git, but a registration key is Ed25519.

## Step 2: sign the registration message

The message is three lines of UTF-8 text, each with a trailing newline:

```text
xhostd-register
<username or empty>
<unix time>
```

The second line is empty when the platform allocates the username. The
namespace of the signature is `xhostd-register`.

```sh
WORK=$(mktemp -d); TS=$(date +%s)
printf 'xhostd-register\n\n%s\n' "$TS" > "$WORK/msg"
ssh-keygen -Y sign -f ~/.ssh/xhost_ed25519 -n xhostd-register "$WORK/msg"
```

`ssh-keygen -Y sign` writes the armored signature to `$WORK/msg.sig`.

To request a name, put it on the second line and send the same value as
`username` in the next step. The two must agree, or the signature does not
verify:

```sh
NAME=agentlisbon7
printf 'xhostd-register\n%s\n%s\n' "$NAME" "$TS" > "$WORK/msg"
ssh-keygen -Y sign -f ~/.ssh/xhost_ed25519 -n xhostd-register "$WORK/msg"
```

A name matches `^agent[a-z0-9]{5,35}$`: the word `agent`, then 5 to 35
lowercase letters or digits. An allocated name is `agent` plus 8 random
characters. The name is part of every hostname of the account, as in
`<app>-<username>.xhostd.app`.

## Step 3: post the registration

The body carries the public key line, the time you signed, and the signature.
The response holds the token, so it goes to a file and never to stdout.

```sh
python3 - "$WORK" "$TS" <<'EOF' > "$WORK/body.json"
import json, pathlib, sys
work, ts = sys.argv[1], int(sys.argv[2])
print(json.dumps({
    "public_key": (pathlib.Path.home() / ".ssh/xhost_ed25519.pub").read_text().strip(),
    "timestamp": ts,
    "signature": pathlib.Path(work, "msg.sig").read_text(),
}))
EOF
curl -sS https://api.xhostd.com/registrations -H 'Content-Type: application/json' \
  --data @"$WORK/body.json" -o "$WORK/response.json" -w 'HTTP %{http_code}\n'
```

Add `"username": "<name>"` to the body when you signed a name, and
`"label": "<text>"` to name the key on the account (64 characters or fewer;
the default label is `registration`). Any other field answers `422`.

The response, with the token shortened:

```json
{
  "user_id": "3f1c2a7e-9b4d-4c1e-8a6f-2d5b7c9e0f13",
  "username": "agent7k2m9x4q",
  "plan": "starter",
  "token": "xh_...",
  "token_expires_at": "2026-10-07T12:00:00Z",
  "ssh_key_id": "b8e4d2c6-1a3f-4e5b-9c7d-0f2a4b6c8e1d",
  "fingerprint_sha256": "SHA256:x4bR5nQm7pZs2tVw9yAcE1gHjK3lMoPqR6sTuVwXyZ0",
  "git_ssh_host": "git.xhostd.com",
  "limits": {
    "tier": "starter", "rank": 0, "max_channels": 1, "cpu_soft_cores": 0.1,
    "cpu_burst": 2, "visible_cores": 1, "mem_limit_mb": 128, "storage_mb": 250,
    "blob_storage_bytes": 134217728, "image_size_bytes": 536870912,
    "snapshot_retention_days": 1, "deploy_snapshot_keep": 1,
    "port_forwarding": false, "agent_registration_only": true
  },
  "next": {
    "verify_email": "POST /me/email-verifications",
    "renew_token": "POST /auth/ssh-key"
  }
}
```

`limits` is the `starter` row of `GET /plans`. `next` names the two routes you
call later: the one that moves the account to `basic`, and the one that
renews the token. `git_ssh_host` is the host of the SSH remote.

## Step 4: store the token

Write the token to `~/.config/xhostd/token` with mode `0600`, and print the
fields that hold no secret:

```sh
python3 - "$WORK/response.json" <<'EOF'
import json, os, pathlib, sys
r = json.load(open(sys.argv[1]))
if "token" not in r:
    print(r); sys.exit(1)          # an error envelope holds no secret
d = pathlib.Path.home() / ".config/xhostd"; d.mkdir(parents=True, exist_ok=True); d.chmod(0o700)
fd = os.open(d / "token", os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
os.fchmod(fd, 0o600)
os.write(fd, r["token"].encode()); os.close(fd)
print({k: r[k] for k in ("username", "user_id", "plan", "token_expires_at", "fingerprint_sha256", "git_ssh_host")})
EOF
rm -rf "$WORK"
```

The file sits in `$HOME`, so every session on the machine reads it, and no
repo holds it. The token never reaches stdout: a value in stdout is a value
in the transcript, and a transcript is a log. The script prints the error
envelope instead when the platform refused the request, because an envelope
holds no secret. Read [Errors](#errors) for what each answer means.

## Step 5: use the token

Write `$(cat ~/.config/xhostd/token)` in a command. Never paste the value.

**The MCP server.** Add it with the token as a header:

```sh
claude mcp add --transport http xhost https://mcp.xhostd.com/mcp/ \
  --header "Authorization: Bearer $(cat ~/.config/xhostd/token)"
```

The tools appear after the client connects again: `/mcp`, then `xhost`, then
reconnect, or a new session. The server name `xhost` is the name the plugin
uses, so the skill's `mcp__xhost__*` names match.

**The HTTP API.** Every route in the
[API reference](https://docs.xhostd.com/api) takes the same header:

```sh
curl -sS https://api.xhostd.com/apps \
  -H "Authorization: Bearer $(cat ~/.config/xhostd/token)"
```

**Git.** A push needs no token. The registration key is registered on the
account already, so skip `register_ssh_key`; it answers `409` for this key,
which is correct. Set the SSH remote and push:

```sh
git remote add xhost-ssh "git@git.xhostd.com:<username>/<app>.git"
GIT_SSH_COMMAND="ssh -i ~/.ssh/xhost_ed25519 -o IdentitiesOnly=yes" git push xhost-ssh HEAD:master
```

[Push code with git](https://docs.xhostd.com/guides/git) explains the
refspec and the HTTPS fallback.

## Renew the token

The token expires 30 days after the mint. Sign a login message and post it
to `POST /auth/ssh-key`. The message is three lines: the word `xhostd-login`,
the key's fingerprint, and the Unix time. The namespace is `xhostd-login`.

```text
xhostd-login
<fingerprint_sha256>
<unix time>
```

The fingerprint is the `SHA256:...` value `ssh-keygen -lf` prints, prefix
included. It equals the `fingerprint_sha256` field of the registration
response.

```sh
FP=$(ssh-keygen -lf ~/.ssh/xhost_ed25519.pub | awk '{print $2}')
WORK=$(mktemp -d); TS=$(date +%s)
printf 'xhostd-login\n%s\n%s\n' "$FP" "$TS" > "$WORK/msg"
ssh-keygen -Y sign -f ~/.ssh/xhost_ed25519 -n xhostd-login "$WORK/msg"
```

Then build the body as in [Step 3](#step-3-post-the-registration), post it to
`https://api.xhostd.com/auth/ssh-key`, and store the token as in
[Step 4](#step-4-store-the-token). The body has the same three fields and
takes no `username` or `label`. The response:

```json
{
  "token": "xh_...",
  "token_expires_at": "2026-11-06T12:00:00Z",
  "user_id": "3f1c2a7e-9b4d-4c1e-8a6f-2d5b7c9e0f13",
  "username": "agent7k2m9x4q"
}
```

A renewal changes the header the MCP client holds, so remove the server and
add it again:

```sh
claude mcp remove xhost
claude mcp add --transport http xhost https://mcp.xhostd.com/mcp/ \
  --header "Authorization: Bearer $(cat ~/.config/xhostd/token)"
```

An account holds at most 20 tokens from this route. A `404` means the
platform holds no key with this fingerprint that can sign in: register the
key with [Step 2](#step-2-sign-the-registration-message) through
[Step 4](#step-4-store-the-token). A key a person registered through
`register_ssh_key` or the console cannot sign in unless it was registered
with `api_login` set.

## Verify an email and move to basic

A `starter` account has no email, so no person can open the console for it.
When a person gives you an address, verify it. The account then moves to
`basic`, and Google sign-in with that address opens the console for this
account.

Two MCP tools do it, and each has a route.

1. Call `request_email_verification` with `email`. The platform mails an
   8-character code to the address and answers `{status: "sent", expires_at}`.
   The code expires 15 minutes after the mail. The code alphabet is lowercase
   letters and digits without `0`, `1`, `i`, `l`, and `o`, so a person can
   read it out with no ambiguity.
2. Ask the person for the code, and call `complete_email_verification` with
   `code`. Success answers `{status: "verified", plan: "basic", apply_queued}`.
   The platform applies the `basic` limits in the background; `apply_queued`
   is `true` when that job is queued now, and `false` when a move holds the
   account and the job runs after it.

The same two calls over HTTP:

```sh
curl -sS https://api.xhostd.com/me/email-verifications \
  -H "Authorization: Bearer $(cat ~/.config/xhostd/token)" \
  -H 'Content-Type: application/json' --data '{"email": "<address>"}'

curl -sS https://api.xhostd.com/me/email-verifications/complete \
  -H "Authorization: Bearer $(cat ~/.config/xhostd/token)" \
  -H 'Content-Type: application/json' --data '{"code": "<code>"}'
```

The rules:

- A second request inside 60 seconds of the last one answers `429`. After
  that window a new request replaces the code, and the old one stops working.
- Five wrong codes lock the challenge, and every later code answers `429`.
  Request a new code after the window.
- An expired code, or no pending request, answers `410`. Request a new code.
- An address that another account verified already answers `409`, and the
  challenge clears. The account keeps `starter`.
- An account that has a verified email answers `409` on the request route.

Once the address is verified, a person who signs in to
[the console](https://console.xhostd.com) with it lands on this account. The
console lists the registration key and the tokens, and the person can revoke
each one there.

## Upgrade

No route lets an agent pay. When a task needs a paid tier, call
`submit_feedback` with the upgrade request, and tell the person that paid
plans are bought in the console after the email is verified.

## Errors

Both anonymous routes answer the standard envelope,
`{"error": {"code": "...", "message": "..."}}`.

| Status | Code | Message | What to do |
|---|---|---|---|
| 400 | `bad_request` | the parser's own message | Send one OpenSSH public-key line, `ssh-ed25519 <blob> [comment]` |
| 400 | `bad_request` | `only ssh-ed25519 keys can register or sign in` | Make an Ed25519 key |
| 400 | `bad_request` | `timestamp is outside the 300-second window; check the clock and sign again` | Fix the clock, then sign a fresh message |
| 400 | `bad_request` | `invalid signature`, `signature namespace mismatch`, or `the signature was made with a different key` | Sign the exact message bytes under the route's namespace with the key you send |
| 400 | `bad_request` | `an agent username is 'agent' followed by 5 to 35 lowercase letters or digits` | Pick a name that matches the rule, or send none |
| 400 | `bad_request` | `the label must hold 64 characters or fewer` | Shorten the label |
| 404 | `not_found` | `no api-login ssh key matches this fingerprint` | Login only: register the key, or use the key that registered |
| 409 | `conflict` | `username is taken` | Pick another name, or send none |
| 409 | `conflict` | `this ssh key is registered already; sign in with it through POST /auth/ssh-key` | Renew the token with this key; never make a second key |
| 409 | `conflict` | `could not allocate a username; retry` | Post the same body again |
| 422 | none | FastAPI's validation body | Read the field name in `detail` and correct the body |
| 429 | `too_many_requests` | `agent registration budget reached (global); retry later` or `(this source)` | The daily budget is spent; retry the next day |
| 503 | `service_unavailable` | `agent registration is closed` | An operator closed registration; a person signs up in the browser instead |

`404` is a login answer. The other rows apply to the registration, and the
400 rows apply to both.

## What the platform stores

- The public key, its `SHA256:` fingerprint, and the label. The private key
  never leaves the machine.
- A keyed hash of the source address prefix, and never the address. The hash
  bounds how many accounts one source opens per day.
- A digest of the token, and never the token. The response is its one copy.
- The verified email, once you verify one. Until then the account holds no
  address.
