# OAuth recipe: how to gate a route on the signed-in visitor

## What you get

A FastAPI app with a public home page and a members-only route. A visitor
signs in with Google through the identity gateway of the platform. The gateway
sets an identity cookie, and the app verifies that cookie. To a visitor who did
not sign in, the app answers 401 with a sign-in link. You register no client
id, you store no secret, you configure no redirect URI and you set no toggle.
Your part is one function that verifies a JWT.

The example app is live at
[recipe-oauth-docs.xhostd.app](https://recipe-oauth-docs.xhostd.app/). The
transcript in this guide deployed the four files below. The app has a public
home page, and a `/private` route that answers 401 until you sign in. If you
obey this recipe on your own account, you get the same app under your own name.

The app runs on the `app` template. Thus `install.sh` and `launch.sh` obey the
contract in
[Recipe: Python API on the app template](https://docs.xhostd.com/guides/recipes-app-python).
This guide adds only the identity layer.

## The platform gates no route for you

This section states the most important fact in this guide: **every request
reaches your app, signed in or not.** The platform does no enforcement at the
edge. The proxy in front of your container removes the platform identity
headers that a caller can spoof. The proxy then sets its own attribution
variables and mounts the `/xhost-auth/*` endpoints. Then the proxy sends every
request to your app. This includes a request for `/private` from a visitor who
never signed in.

An app can add a `/private` route and assume that the platform protects it.
That app has an open endpoint. The `current_user()` function below is the only
control between an anonymous caller and a gated route. If you delete the
function, no other control replaces it.

This design has a benefit: the platform never guesses your access policy. The
platform gives you a verified identity, and you decide what that identity can
do. You can decide per route, per record, or per any other unit.

## The sign-in sequence

Six endpoints are available under `/xhost-auth/` on every channel hostname. A
verified custom domain also has them. You do not deploy these endpoints. The
platform mounts them in front of your app.

| Endpoint | What it does |
|---|---|
| `/xhost-auth/login` | Mints a short-lived state token and 302s to Google. Accepts `return_to`. |
| `/xhost-auth/callback` | Google's redirect target, on `auth.xhostd.com`. Exchanges the code, mints a 60-second transfer token, 302s to your channel's `/finalize`. |
| `/xhost-auth/finalize` | Verifies the transfer token, sets the identity cookie, 302s to `return_to`. |
| `/xhost-auth/logout` | Clears the cookie. Accepts `return_to`. |
| `/xhost-auth/whoami` | JSON identity probe. Always 200; useful from JavaScript on a static page. |
| `/xhost-auth/jwks` | The public keys your app verifies the cookie against. |

The `login`, `finalize` and `logout` endpoints accept `return_to`. The value
must be a single local absolute path. If the value can redirect the visitor off
the channel, the gateway uses `/` instead. Send a visitor to
`/xhost-auth/login?return_to=/private`. The gateway then returns the visitor to
the gated page after the sign-in.

You configure nothing. The platform owns the Google project, the consent screen
and the signing key. When you created the app, you completed all the necessary
steps.

## The identity cookie

`/finalize` sets one cookie with the name `__Host-xhost_id`. The value of the
cookie is an RS256 JWT.

| Attribute | Value |
|---|---|
| `HttpOnly` | yes — JavaScript cannot read it |
| `Secure` | yes |
| `SameSite` | `lax`, so the top-level redirect back from Google carries it |
| `Path` | `/` |
| `Max-Age` | 86400, one day |
| `Domain` | absent |

The browser enforces the `__Host-` prefix. A browser accepts a cookie with that
prefix only if the cookie is `Secure`, has `Path=/`, and has no `Domain`. A
cookie with no `Domain` is host-only. The browser sends it back only to the
exact hostname that set it. Thus the browser never offers a token from one
channel to a different channel.

The payload carries seven claims:

| Claim | Meaning |
|---|---|
| `iss` | `https://auth.xhostd.com` |
| `aud` | **the channel's own hostname** |
| `sub` | Google's stable user id |
| `email` | the verified Google address |
| `name` | display name |
| `iat` | issued at |
| `exp` | `iat` + 86400 |

Two of these claims need care.

**The gateway derives `iss`; it is not a constant.** The gateway builds the
value as `f"https://auth.{settings.domain}"`. Thus `https://auth.xhostd.com` is
the correct literal for production. The value comes from the configured
platform domain, not from a hardcoded string. Pin `iss` in your verification,
and remember its source.

**`aud` is your channel's hostname, and no component tells your container that
name.** No environment variable carries it. The correct source is the request.
The proxy passes the browser's `Host` header through without a change. Thus
`request.url.hostname` is the audience of the token. The same code stays
correct when you point a custom domain at the channel, because the cookie on
that domain carries *that* hostname as `aud`. If you hardcode the
`*.xhostd.com` name, the app fails as soon as a visitor arrives on a custom
domain. The app then reports every signed-in visitor as signed out.

## The files

Four files at the repo root.

### app.py

```python
import logging
from typing import Any

import jwt
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse, JSONResponse

# uvicorn's logger, so these lines show up in the channel's logs.
logger = logging.getLogger("uvicorn.error")

COOKIE_NAME = "__Host-xhost_id"
ISSUER = "https://auth.xhostd.com"
LOGIN_URL = "/xhost-auth/login?return_to=/private"

# One JWKS fetch per process instead of one per request: PyJWKClient keeps the
# key set in memory and re-fetches it when a token turns up with a `kid` it
# has not seen, which is how a platform key rotation is picked up.
_jwks = jwt.PyJWKClient(f"{ISSUER}/xhost-auth/jwks", timeout=5)

app = FastAPI()

_SIGNED_OUT = f"""<!doctype html>
<html lang="en">
<meta charset="utf-8">
<title>oauth-gated</title>
<h1>oauth-gated</h1>
<p>You are not signed in.</p>
<p><a href="{LOGIN_URL}">Sign in with Google</a></p>
</html>"""

_SIGNED_IN = """<!doctype html>
<html lang="en">
<meta charset="utf-8">
<title>oauth-gated</title>
<h1>oauth-gated</h1>
<p>You are signed in.</p>
<p><a href="/private">Members-only page</a></p>
<p><a href="/xhost-auth/logout?return_to=/">Sign out</a></p>
</html>"""


def current_user(request: Request) -> dict[str, Any] | None:
    """Return the verified identity claims, or None if not signed in.

    The platform does NO edge enforcement: every request reaches this app
    whether or not the visitor signed in, so this function is the only thing
    standing between an anonymous caller and a gated route.
    """
    token = request.cookies.get(COOKIE_NAME)
    if not token:
        return None
    try:
        # Selects the key by the token header's `kid`.
        signing_key = _jwks.get_signing_key_from_jwt(token)
        return jwt.decode(
            token,
            signing_key.key,
            # Pinned: never let the token's own `alg` header pick the
            # algorithm, or a forged token can pick one you did not intend.
            algorithms=["RS256"],
            issuer=ISSUER,
            # `aud` is this channel's own hostname. Nothing injects that name
            # into the container, and the platform passes the browser's Host
            # header through untouched, so the request's host is the source —
            # and it stays correct for a custom domain too.
            audience=request.url.hostname,
            leeway=60,
            options={
                "require": ["exp", "iss", "aud", "sub", "email"],
                "strict_aud": True,
            },
        )
    except jwt.PyJWTError as exc:
        # Anything unverifiable — bad signature, wrong audience, expired,
        # unreachable JWKS — is treated as "not signed in".
        logger.warning("identity cookie rejected: %s", exc)
        return None


# The health check probes GET / and needs a 2xx.
@app.get("/")
def home(request: Request):
    # Both pages are fixed HTML. Profile fields are attacker-influenced, so
    # they go out as JSON below rather than interpolated into markup.
    return HTMLResponse(
        _SIGNED_IN if current_user(request) is not None else _SIGNED_OUT
    )


@app.get("/private")
def private(request: Request):
    user = current_user(request)
    if user is None:
        return JSONResponse(
            {"error": "Sign in to see this page.", "login_url": LOGIN_URL},
            status_code=401,
        )
    # `sub` is Google's stable user id — key your own records on it, because
    # `email` can change.
    return {"sub": user["sub"], "email": user["email"], "name": user.get("name")}
```

Five details in that file are important.

**Pin `algorithms=["RS256"]`. This is not a formality.** A JWT carries its own
`alg` header. A verifier that trusts that header lets the attacker select how
the verifier checks the token. The attacker sets `alg: none` to remove the
signature check. Or the attacker sets a symmetric algorithm, and the verifier
uses the *public* key as an HMAC secret. The platform publishes that public key
at a URL that anyone can read. Both attacks make a published public key into a
key that forges tokens. Give the library the list of algorithms that *you*
accept, and the library ignores the `alg` header of the token.

For the same reason, never use a decode-only function. Some libraries supply a
function that parses a JWT but does not verify the signature. PyJWT has one:
`jwt.decode(..., options={"verify_signature": False})`. Such a function is for
a token that you already trust. It is not for a visitor. An identity check that
uses no key is not a check.

**`PyJWKClient` selects the key by the `kid` in the token header, and keeps the
key set in memory.** The client fetches the key set again when the cache
lifetime expires, 300 s by default. The client also fetches again immediately
when a token carries a `kid` that it has not seen. A key rotation on the
platform appears as an unseen `kid`, so your app gets the new key with no
redeploy. Your app can fetch the JWKS on each request instead. That method also
works, but it is slower and it fails more often.

**`audience=request.url.hostname`, with `strict_aud`.** The audience comes from
the request, for the reason above. `strict_aud` refuses the loose match that a
list value for `aud` permits. The `require` option makes the library reject a
token that has no `exp`, `iss`, `aud`, `sub` or `email` claim.

**`leeway=60`.** The verification accepts 60 seconds of clock difference
between the host that signs the token and your host. Without the leeway, your
app can reject a new token because its `iat` claim is in the future.

**Every failure means "not signed in".** A bad signature, a wrong audience, an
expired token and an unreachable JWKS all raise `jwt.PyJWTError`, and
`current_user` returns `None`. This is the safe direction of failure. The
visitor sees the signed-out page, and the app never accepts a token that it
cannot verify. The example logs the rejection with the true reason, so you can
tell an expiry from an outage.

Also examine what the handlers do with the claims. `home` returns fixed HTML,
and it never puts a profile field into the markup. The `name` and `email`
claims come from a third party, and an attacker can influence them. `private`
returns the claims as JSON, and the encoder escapes them. Use `sub` to identify
the user, not `email`, because `sub` is the stable identifier from Google. An
email address can move to a different person.

### requirements.txt

```text
fastapi==0.115.6
uvicorn==0.34.0
PyJWT[crypto]==2.13.0
cryptography==49.0.0
```

`PyJWT[crypto]` is the important line. PyJWT alone cannot do RS256. The
`[crypto]` extra installs `cryptography`, which supplies the RSA code. If you
install plain `PyJWT`, every verification fails with an error, because a
dependency is not present. That error occurs at request time, not at start-up.
The file also pins `cryptography` directly, so the transitive version cannot
change.

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

No step here is specific to OAuth. The sign-in works for the whole platform.
The identity gateway is already in front of every app on every channel
hostname. Thus you enable no per-app toggle, and you register no client id and
no secret. Every step in this sequence is also in the other recipes.

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
create_app(name="recipe-oauth", template="app")
→ {"id": "4d06ab64-0e2e-4ed9-96b0-51450fd673c8",
   "name": "recipe-oauth",
   "template": "app",
   "repo_url": "https://git.xhostd.com/docs/recipe-oauth.git",
   "channels": [{"id": "2997a56a-b7a4-4f9a-992c-2a83ff40578e",
                 "name": "prod",
                 "hostname": "recipe-oauth-docs.xhostd.app",
                 "current_sha": null}], ...}
```

Keep the app `id` and the `prod` channel `id`, because every later call takes
one or both. Also record the hostname in the response. That hostname is the
`aud` claim in your tokens.

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
$ git clone https://docs:$XHOST_TOKEN@git.xhostd.com/docs/recipe-oauth.git
Cloning into 'recipe-oauth'...
warning: You appear to have cloned an empty repository.
```

Write the four files into that directory. Then commit and push them:

```bash
$ cd recipe-oauth
$ git add -A
$ git commit -m "oauth-gated recipe"
$ git push origin master
```

Do not put the token in a file that you commit, and do not paste it.
`$XHOST_TOKEN` above is the value from `get_credentials`. A remote URL with a
real token goes into `.git/config`, into your shell history, and into the
output of every `git remote -v`.

**4. Deploy the branch.**

```text
deploy(app_id="4d06ab64-0e2e-4ed9-96b0-51450fd673c8",
       channel_id="2997a56a-b7a4-4f9a-992c-2a83ff40578e",
       ref="master")
→ {"deploy_id": "c141a3e1-00e0-466f-b45d-aad8968edc3f",
   "channel_id": "2997a56a-b7a4-4f9a-992c-2a83ff40578e",
   "status": "queued"}
```

`ref` is a branch name, and xhostd resolves it to the current head of that
branch. After a push, you do not need to know the sha. `deploy` also accepts
`sha` when you want an exact commit, and `sha` has priority if you give both.
The deploy then runs in the background. Use `get_deploy_log` to follow it.

## Verify it

### The deploy log

This is the deploy above, `c141a3e1-00e0-466f-b45d-aad8968edc3f`. The log below
has no buildkit lines and no uvicorn start-up lines, because they teach
nothing.

```text
...
[2026-07-31T18:25:57+00:00] [build] #7 0.814  + cryptography==49.0.0
[2026-07-31T18:25:57+00:00] [build] #7 0.815  + pyjwt==2.13.0
...
[2026-07-31T18:25:59+00:00] [build] image 982.32 MB total, 33.67 MB charged — base xhost-runtime:node22-py313 exempt
[2026-07-31T18:26:02+00:00] health_check container=696e75f2fe6c85c63f4fa7879f5a4247aabc45cec5414d5618fc90cc29f4b11a port=3000 timeout=120.0s
[2026-07-31T18:26:02+00:00] [container] [xhost] starting launch.sh (XHOST_HTTP_PORT=3000) ...
[2026-07-31T18:26:04+00:00] health_check ok
[2026-07-31T18:26:04+00:00] [container] INFO:     Uvicorn running on http://0.0.0.0:3000 (Press CTRL+C to quit)
[2026-07-31T18:26:04+00:00] [container] INFO:     10.77.1.5:54854 - "GET / HTTP/1.1" 200 OK
[2026-07-31T18:26:05+00:00] caddy ensure_route hostname=recipe-oauth-docs.xhostd.app upstream=10.77.1.5:32046
[2026-07-31T18:26:05+00:00] deploy success
```

Read two lines in that log.

**`+ cryptography==49.0.0` and `+ pyjwt==2.13.0`.** One `PyJWT[crypto]==2.13.0`
line in `requirements.txt` installs two distributions. The `cryptography` line
in the build log confirms that RS256 works. If the build log has `pyjwt` but no
`cryptography`, your `requirements.txt` does not have the `[crypto]` extra.

**`health_check ok`, and the `"GET / HTTP/1.1" 200 OK` that caused it.** The
probe requests `/` with no identity. The lines from the container reach the
deploy log later than the lines from the platform. Thus the 200 line prints
after the `health_check ok` line that it caused. The probe passes because the
home page is public. A root route that redirects an anonymous caller to the
sign-in returns a 302, and the probe accepts a 302. A root route that answers
401 fails the deploy.

### The anonymous side

The two calls below are anonymous. They send no cookie, and nothing in the
request identifies the caller. Both outputs come from the live channel.

```console
$ curl -sS -o /dev/null -w '%{http_code}\n' https://recipe-oauth-docs.xhostd.app/
200

$ curl -sS -w ' [%{http_code}]\n' https://recipe-oauth-docs.xhostd.app/private
{"error":"Sign in to see this page.","login_url":"/xhost-auth/login?return_to=/private"} [401]
```

The home page answers 200 to an unknown visitor. The body is the `_SIGNED_OUT`
page from the source above: "You are not signed in.", with a link to
`/xhost-auth/login?return_to=/private`. That `return_to` value returns the
visitor to the gated page after Google completes the sign-in.

Examine the 401 with care. The request *reached the app*, and the app refused
it. No component upstream stopped the request. `current_user` found no cookie
and returned `None`, and the handler selected the status. An absent cookie, a
forged cookie, an expired cookie and a wrong audience all give this same
response, because `current_user` turns every `jwt.PyJWTError` into `None`. The
response tells the caller only to sign in.

### The signed-in side, and why it is not here

This guide has no transcript of an authenticated request. A Google sign-in
needs a browser: a consent screen, a sequence of redirects, and a cookie that
the browser stores. `curl` cannot do that. Thus the comparison above shows only
one half. An invented cookie or a made-up response body would be worse than
this admission.

The handlers of the example decide the result of a signed-in request, and you
can read that result in `app.py` above. `current_user` returns the verified
claims, not `None`. Thus `home` serves `_SIGNED_IN`: "You are signed in.", a
link to `/private`, and a sign-out link to `/xhost-auth/logout?return_to=/`.
`private` no longer answers 401. It returns three fields from those claims as JSON:
`sub`, `email` and `name`. A real body therefore holds a real Google user id
and a real address. That is the second reason why this public document prints
no such body.

Run the anonymous half against your own app. That check finds the first failure
in the next section, and it needs one command.

## When it goes wrong

### The route is open, and nobody noticed

This failure has no error message, and that makes it the worst failure in this
list. The symptom is a `/private` route that answers 200 to a `curl` call with
no cookie. Test each route that you think is gated in that way, every time:

```console
$ curl -sS -o /dev/null -w '%{http_code}\n' https://<your-host>/private
```

A 200 means that you serve that page to the internet. The platform has no
enforcement at the edge to protect you.

### Every signed-in visitor looks signed out

The audience is almost always the cause. Your app can hardcode its
`*.xhostd.com` hostname as the `audience`. If the visitor arrives on a custom
domain, the `aud` claim in the cookie is the custom domain. The verification
then fails on every request, also on the first request after a correct sign-in.
The app looks as if the gateway never sets the cookie. Take the audience from
`request.url.hostname`, and this failure cannot occur.

The log line from the example identifies the cause.
`identity cookie rejected: Audience doesn't match` is this failure.
`Signature has expired` is an old cookie, and that is not a fault.

### The cookie name is wrong

The cookie name is `__Host-xhost_id`, with the prefix.
`request.cookies.get("xhost_id")` returns `None` for every visitor, so the app
reports all visitors as signed out. The `__Host-` prefix is part of the name
that the browser stores and sends. It is not decoration.

### You installed `PyJWT` without the `crypto` extra

The build succeeds and the app starts. Then every verification fails at request
time with an error that names RS256 as an unsupported algorithm. Look for
`+ cryptography` in the build log. If that line is absent, `requirements.txt`
says `PyJWT` in place of `PyJWT[crypto]`.

### The verification does not verify

A call that decodes a token with no key accepts a token that anyone can mint. A
call that gives the token's own header algorithm to the verifier does the same.
An app that catches the verification error and then uses the unverified payload
also does the same. Your check must fetch a key. Your check must pin
`algorithms=["RS256"]`. Your check must treat every exception as "not signed
in".

### The JWKS endpoint is unreachable

`PyJWKClient` cannot fetch the key set, and the verification raises an error.
The example then reports every visitor as signed out until the endpoint
recovers. This is the correct direction of failure. But an app that never
fetched the JWKS looks like a broken login, not like a network fault. The
reason in the log separates the two.

### You keyed your records on `email`

A Google address can change. `sub` is the stable identifier, and the rows of a
user must point at `sub`. Store `email` with it as a display value. Update
`email` from each new token.

### The health check fails because `/` requires sign-in

The probe requests `/` with no cookie, and it needs a 2xx or a 3xx. A root
route that answers 401 to an anonymous caller fails the deploy, although the
app runs correctly. Keep a public home page, as the example does. As an
alternative, redirect an anonymous caller to `/xhost-auth/login`. That redirect
is a 302, so the probe accepts it.

A page can have no server-side code and still need the visitor's identity. The
`/xhost-auth/whoami` endpoint answers JSON to a `fetch()` from a
[static site](https://docs.xhostd.com/guides/recipes-static). The endpoint
always answers 200. It carries `logged_in`, and it adds a `login_url` and a
`logout_url` when the visitor is anonymous.
