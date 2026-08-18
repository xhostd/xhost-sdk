# Push code with git

Every app on xhost owns a git repo at
`https://git.xhostd.com/<username>/<app>.git`. A push to that repo puts
code onto an app. This applies to the first commit, and to every commit
after it. A push sends only the diff. Thus each edit stays small, and it
costs much less than a full file through an MCP tool call. A push stores
your code; it does not deploy the code. Start a build separately with
`deploy`, or with the console.

## Pick the path before you push

The repo answers two transports, and one MCP tool writes to it directly.
Test the machine first, and pick one of the three paths below. Do not
wait for a failure.

1. **The machine runs no command.** A runtime with no shell, such as the
   claude.ai connector, cannot run git. Use the `commit_files` MCP tool.
2. **The machine runs commands.** Push over SSH. Read **Push over SSH**
   below. Reuse `~/.ssh/xhost_ed25519` if that file exists. If the file
   is absent, make a keypair and call `register_ssh_key` once.
3. **The network blocks outbound port 22, or an SSH push fails.** Push
   over HTTPS with a token in the URL. Read **The clone / push URL**
   below.

SSH is the first transport because a private key never enters a tool
call. An HTTPS remote holds a 30-day token, and a content filter in your
AI tool can refuse a tool call that holds a secret. Such a refusal is
not a clean error that your tool can act on. Your tool can stop, or it
can ask the person to run a command by hand. One extra step at the start
is better than that risk.

The SSH path costs no extra step in the steady state. Your tool makes a
keypair only when the key file is absent. A key belongs to the account
and not to one app, so one `register_ssh_key` call serves every app on
that machine.

## Mint credentials

You mint one **unified credential**. A single `xh_` token is your git
password over HTTPS, your Postgres password, and your platform API
bearer. It carries the full default scopes: repo, deploy, channel, db
and blob. Thus the same token can push code, deploy code, and manage
your apps. An SSH push needs no token; read **Push over SSH** below.

- **From your AI tool:** call the `get_credentials` MCP tool. It
  returns `{token, username, expires_at, scopes}`.
- **From the console:** open
  [console.xhostd.com/tokens](https://console.xhostd.com/tokens) and
  create a token.

The token starts with `xh_` and is valid for **30 days**. After it
expires, mint a new token.

## Push over SSH

The git host answers SSH, and both transports reach the same repo. This
is the first path on a machine that runs commands. A private SSH key
never enters a tool call, so a content filter has nothing to refuse.

**An SSH push needs no token.** It needs your username and the app name
only. `get_app` and `GET /apps/{app_id}` return the `repo_url` of each
app, and that URL holds both values.

Read `~/.ssh/xhost_ed25519` first. If that file exists, the machine
holds a key already, and you go straight to the push step below.

**Always use the path `~/.ssh/xhost_ed25519`.** Do not put the key in
the project directory. Do not add a suffix for a project, an app, or a
tool. The path is in your home directory, so every tool and every
project on the machine reads the same key.

A different path makes a second keypair. Each new keypair adds a key to
your account, and the platform sends you a notification. Your account
then collects keys that no tool uses.

If the file is absent, make a keypair on the machine that pushes:

```sh
ssh-keygen -t ed25519 -N "" -f ~/.ssh/xhost_ed25519
```

That command writes the private half straight to the disk. Send the
**public** half only — the content of `~/.ssh/xhost_ed25519.pub`. Call
the `register_ssh_key` MCP tool with that content. Add a `label` that
identifies the machine. You can also paste that content on
[console.xhostd.com/ssh-keys](https://console.xhostd.com/ssh-keys), which
lists the keys of your account and deletes one.

**A key belongs to the account, not to one app.** Thus one
`register_ssh_key` call serves every app that you push to from that
machine. Repeat this step on a new machine only.

Then set the SSH remote and push:

```sh
git remote add xhost-ssh "git@git.xhostd.com:<username>/<app>.git"
GIT_SSH_COMMAND="ssh -i ~/.ssh/xhost_ed25519" git push xhost-ssh HEAD:master
```

The `HEAD:master` refspec is deliberate. xhost binds the prod channel to
the `master` branch. A new local repo often uses `main` as its default
branch. `HEAD:master` pushes the current branch to `master`, whatever
its local name is. `GIT_SSH_COMMAND` names the private half. The key
sits outside a default path, so every SSH push needs that variable.

`list_ssh_keys` returns the keys of your account, newest first. It
returns metadata only: `id`, `label`, `algo`, `fingerprint`,
`created_at` and `last_used_at`. It never returns a key.
`delete_ssh_key` removes one key by `key_id`, and a push with that key
then fails at once.

A fingerprint is unique over the whole platform. Thus a key that the
platform holds already answers `409`. For the key at
`~/.ssh/xhost_ed25519`, a `409` means an earlier session registered that
same key. The key works, so push with it. Never make a second keypair to
clear a `409`.

An SSH push needs outbound port 22. Some company networks and CI runners
block that port. Two conditions send you to a different path: the
network blocks port 22, or an SSH push fails. Push over HTTPS then, as
the next section describes, or use the `commit_files` tool.

## The clone / push URL

This is the HTTPS transport. Push over HTTPS where the network blocks
outbound port 22, or where an SSH push fails.

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

The `HEAD:master` refspec is deliberate here too. xhost binds the prod
channel to the `master` branch, and a new local repo often uses `main`
as its default branch.

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

Of the two HTTPS forms, the Basic-password form above is the usual one,
because git's credential helpers use it. Use the Bearer header only if
your tools already add an `Authorization` header.

## Renew an expired token

When a token expires, a push returns `401`. Mint a new token with
`get_credentials`, or with the console. Then update the remote URL:

```sh
git remote set-url xhost "https://<username>:<new-token>@git.xhostd.com/<username>/<app>.git"
```

## A push fails again and again: clear a stale cached credential

Over HTTPS, git keeps the credentials in a **credential helper**. The
helper is the macOS Keychain, `libsecret` on Linux, the Windows
Credential Manager, or a plain `~/.git-credentials` file. This section
applies to the HTTPS path alone, because an SSH push reads no helper.
If a helper holds an old or wrong password for `git.xhostd.com`, git
sends that password again at each push. Your pushes then fail with
`401` or `403`, even after you mint a new token. git does not ask you
again for a credential, because it has one already.

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

- Over HTTPS, git.xhostd.com authenticates you with **Basic auth and the
  token as the password**. git's credential helpers use this form.
  git.xhostd.com also accepts an `Authorization: Bearer` header, as
  shown above. The same token works on the REST API (`api.xhostd.com`)
  and on the git host. Over SSH the key alone authenticates you.
- A `401` means that there is no valid token; authenticate again. A
  `403` means that the token is valid, but it gives no access to that
  repo.
- The username in the URL path only locates the repo. The token's role
  on the app decides your access. Thus you can push to every app that
  you own, and to every app where you are a member.
