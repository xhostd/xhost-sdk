# Push code with git

Every app on xhost owns a git repo at
`https://git.xhostd.com/<username>/<app>.git`. A push to that repo over
plain HTTPS is the standard method to put code onto an app. This applies
to the first commit, and to every commit after it. A push sends only the
diff. Thus each edit stays small, and it costs much less than a full
file through an MCP tool call. Your AI tool also has a `commit_files`
tool. Use that tool in one case only: git is not available on the
machine that the tool uses.

You do not make SSH keys, and you do not keep a separate git password.
You mint a short token, put the token in the URL, and then push. A push
stores your code; it does not deploy the code. Start a build separately
with `deploy`, or with the console.

## Mint credentials

You mint one **unified credential**. A single `xh_` token is your git
password, your Postgres password, and your platform API bearer. It
carries the full default scopes: repo, deploy, channel, db and blob.
Thus the same token can push code, deploy code, and manage your apps.

- **From your AI tool:** call the `get_credentials` MCP tool. It
  returns `{token, username, expires_at, scopes}`.
- **From the console:** open
  [console.xhostd.com/tokens](https://console.xhostd.com/tokens) and
  create a token.

The token starts with `xh_` and is valid for **30 days**. After it
expires, mint a new token.

## The clone / push URL

Put the token in the **password** field of the URL. This is the one
important detail:

```
https://<username>:<token>@git.xhostd.com/<username>/<app>.git
```

The git host ignores the username field, and checks the password only.
Thus any username works. The GitHub form is equivalent, and it makes
the same point clear:

```
https://x-access-token:<token>@git.xhostd.com/<username>/<app>.git
```

Set the remote and push:

```sh
git remote add xhost "https://<username>:<token>@git.xhostd.com/<username>/<app>.git"
git push xhost HEAD:master
```

The `HEAD:master` refspec is deliberate. xhost binds the prod channel to
the `master` branch. A new local repo often uses `main` as its default
branch. `HEAD:master` pushes the current branch to `master`, whatever
its local name is.

Use `git remote set-url xhost ...` if the remote already exists. `get_app`
and `GET /apps/{app_id}` return the exact `repo_url` of each app.

### Interactive form (no token in the URL)

If you do not want the token in the remote, clone with a plain URL. git
then asks you for the credential:

```sh
git clone https://git.xhostd.com/<username>/<app>.git
```

git asks for a username; type any value. git then asks for a password;
put the token at the **Password** prompt.

### Bearer header (advanced)

git.xhostd.com also accepts the token as a Bearer header. The REST API
and the MCP tools present the token in the same way. Thus one credential
is the same on every surface:

```sh
git config http.extraHeader "Authorization: Bearer <token>"
```

The Basic-password form above is the primary path, because git's
credential helpers use it. Use the Bearer header only if your tools
already add an `Authorization` header.

## Renew an expired token

When a token expires, a push returns `401`. Mint a new token with
`get_credentials`, or with the console. Then update the remote URL:

```sh
git remote set-url xhost "https://<username>:<new-token>@git.xhostd.com/<username>/<app>.git"
```

## A push fails again and again: clear a stale cached credential

git keeps the credentials in a **credential helper**. The helper is the
macOS Keychain, `libsecret` on Linux, the Windows Credential Manager, or
a plain `~/.git-credentials` file. If a helper holds an old or wrong
password for `git.xhostd.com`, git sends that password again at each
push. Your pushes then fail with `401` or `403`, even after you mint a
new token. git does not ask you again for a credential, because it has
one already.

Find which helper is active:

```sh
git config --get credential.helper
```

Then clear the stale entry for `git.xhostd.com`:

- **`store` helper** — edit `~/.git-credentials`, and delete the line
  with `git.xhostd.com` in it.
- **macOS Keychain** — open *Keychain Access*, and remove the
  `git.xhostd.com` internet-password entry. You can also use
  `git credential-osxkeychain erase`.
- **Any helper** — tell git to remove it:

  ```sh
  printf 'protocol=https\nhost=git.xhostd.com\n\n' | git credential reject
  ```

The next push then asks you for a new token, or reads the token from the
URL.

## Notes

- The primary authentication on git.xhostd.com is **Basic auth with the
  token as the password**. git's credential helpers use this form.
  git.xhostd.com also accepts an `Authorization: Bearer` header, as
  shown above. The same token works on the REST API (`api.xhostd.com`)
  and on the git host.
- A `401` means that there is no valid token; authenticate again. A
  `403` means that the token is valid, but it gives no access to that
  repo.
- The username in the URL path only locates the repo. The token's role
  on the app decides your access. Thus you can push to every app that
  you own, and to every app where you are a member.
