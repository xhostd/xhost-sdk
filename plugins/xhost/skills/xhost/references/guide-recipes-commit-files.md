# Recipe: ship without git

## What you get

You get the same kind of static site as
[Recipe: static site](https://docs.xhostd.com/guides/recipes-static). Here
`commit_files` puts the whole site onto the app. `commit_files` is a tool
call with a `{path: contents}` map. The platform turns that map into one
ordinary commit on the app's own repo. There is no shell, no clone and no
push.

**`git push` and then `deploy` is the standard path, and this recipe is not
that path.** Use this recipe in one case only: git is not available. That
case is a runtime with no shell, such as the claude.ai connector, a Custom
GPT, or an MCP client that cannot run a command. If you can run `git` on the
machine where you work, obey
[Recipe: static site](https://docs.xhostd.com/guides/recipes-static) instead.
Its four steps are the same on every template.

`commit_files` is not a lesser mode: it writes real commits with your name on
them. They go into the same repo that a push goes to, and a later
`git clone` shows them like any other commits. The cost is per edit. A
changeset sends the full text of every file that it names through the model's
context, and it does this each time. A push sends only the diff. On a
one-file first commit the two costs are about equal. At the tenth edit the
push is a few lines, and the changeset is the whole project again. That is
why git is always the first choice, whatever the size of the project.

The worked example is live at
[recipe-commit-files-docs.xhostd.app](https://recipe-commit-files-docs.xhostd.app/).
It is the two files below, and the transcript in this guide deployed them.
Git did not run in that transcript, on any machine, at any point.

## The files

There are two files at the repo root: the page, and the stylesheet that the
page links to. A later changeset names neither file, and the platform then
leaves the stylesheet as it is.

### index.html

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Shipped without git — xhostd recipe</title>
<link rel="stylesheet" href="/style.css">
</head>
<body>
<main>
  <h1>Shipped without git.</h1>
  <p>
    Every file on this page arrived through <code>commit_files</code> — a
    single tool call carrying a <code>{path: contents}</code> map, which the
    platform turns into one ordinary commit on the app's own repo.
  </p>
  <p>
    That is the fallback path. When git is available, <code>git push</code>
    is the one to use: it sends only what changed, so the tenth edit costs a
    few lines instead of the whole project.
  </p>
  <p><a href="https://docs.xhostd.com/guides">xhostd recipes</a></p>
</main>
</body>
</html>
```

### style.css

```css
:root { color-scheme: light dark; }
body {
  margin: 0; min-height: 100vh;
  display: grid; place-items: center;
  font: 16px/1.6 ui-sans-serif, system-ui, -apple-system, "Segoe UI", sans-serif;
  background: #0b0d12; color: #e6e9ef;
}
main { max-width: 34rem; padding: 2rem; }
h1 { font-size: 1.75rem; margin: 0 0 .5rem; letter-spacing: -.02em; }
p { margin: 0 0 1rem; color: #9aa4b8; }
code {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: .9em; background: #171b24; padding: .15em .4em; border-radius: 4px;
}
a { color: #7aa2f7; }
```

The `static` template serves `index.html` from the repo root at `/`. It
serves `style.css` at `/style.css`, which is the path in the page's `<link>`.
The template serves each committed file at its equivalent path, and it does
nothing else.

## The deploy

This path keeps two acts separate, exactly as git does.
**`commit_files` stores your code, but it does not deploy your code.**
`deploy` is its own explicit call, and it takes the `sha` from the changeset.
This transcript shows that separation: it makes three commits, and it deploys
two of them.

Do these five steps in order.

**1. Create the app.** This call makes the `prod` channel, its hostname and
its git repo. You get a repo even if you never push to it.

```text
create_app(name="recipe-commit-files", template="static")
→ {"id": "b882c472-2a7b-414a-994b-d9c7c3bcdcee",
   "name": "recipe-commit-files",
   "template": "static",
   "repo_url": "https://git.xhostd.com/docs/recipe-commit-files.git",
   "channels": [{"id": "17a905c3-9365-4c1e-be98-7823f83a2647",
                 "name": "prod",
                 "hostname": "recipe-commit-files-docs.xhostd.app",
                 "current_sha": null}], ...}
```

Keep two ids: the app's `id`, and the `id` of the new `prod` channel. Every
later call needs one or both. This recipe has no credential step.
`commit_files` uses the same authenticated tool session as every other call.
You thus make no token, store no token and paste no token. That is the one
advantage of the fallback path over the standard path.

**2. Commit the whole site in one changeset.** The `files` map is
`{path: contents}`. A string value writes the file at that path, and it
replaces the file if the file exists.

```text
commit_files(app_id="b882c472-2a7b-414a-994b-d9c7c3bcdcee",
             message="the whole site, in one changeset",
             files={"index.html": "<!doctype html>\n<html lang=\"en\">\n...",
                    "style.css": ":root { color-scheme: light dark; }\n..."})
→ {"sha": "0bf2770b0ab04b119bc4138faf2d9b1a270ab095"}
```

The transcript above shortens the two values. In the real call each value is
the full text of its file from the section above, with all the newlines.
That is the cost:
both files went through the model's context in full, for one commit.

`list_files` reads the branch back. Call it after every changeset until this
path is familiar to you:

```text
list_files(app_id="b882c472-2a7b-414a-994b-d9c7c3bcdcee")
→ {"ref": "master", "sha": "0bf2770b0ab04b119bc4138faf2d9b1a270ab095",
   "files": [{"path": "index.html", "kind": "blob", "size": 815},
             {"path": "style.css", "kind": "blob", "size": 563}]}
```

**3. Deploy that commit.**

```text
deploy(app_id="b882c472-2a7b-414a-994b-d9c7c3bcdcee",
       channel_id="17a905c3-9365-4c1e-be98-7823f83a2647",
       sha="0bf2770b0ab04b119bc4138faf2d9b1a270ab095")
→ {"deploy_id": "df8750a5-ada3-45fe-b3b1-1543ec14873d",
   "channel_id": "17a905c3-9365-4c1e-be98-7823f83a2647",
   "status": "queued"}
```

Use `sha` on this path, not `ref`. `commit_files` returns the exact commit
that it wrote, so you deploy that commit. `ref="master"` is also valid, and it
resolves to the branch's current head. `sha` wins if you give both. `deploy`
returns as soon as it queues the deploy, and the work runs asynchronously.
Follow it with `get_deploy_log`.

**4. Change one file, and only that file.** The platform does not change a
path that the map does not name. This changeset names one new path:

```text
commit_files(app_id="b882c472-2a7b-414a-994b-d9c7c3bcdcee",
             message="add robots.txt",
             files={"robots.txt": "User-agent: *\nDisallow:\n"})
→ {"sha": "9664363b9339eec27087c819f49c14c977799134"}
```

```text
list_files(app_id="b882c472-2a7b-414a-994b-d9c7c3bcdcee")
→ {"ref": "master", "sha": "9664363b9339eec27087c819f49c14c977799134",
   "files": [{"path": "index.html", "kind": "blob", "size": 815},
             {"path": "robots.txt", "kind": "blob", "size": 24},
             {"path": "style.css", "kind": "blob", "size": 563}]}
```

**That list is the reason for this step.** The changeset named `robots.txt`
and no other path. `index.html` and `style.css` did not change, and they keep
the same sizes as after commit 1: 815 and 563 bytes. A changeset is a sparse
patch, not a full picture of the tree. The platform applies it on top of the
branch's current head. **If you omit a file, the platform leaves that file as
it is. It never deletes the file.** Send only the paths that change.

**5. Delete it again.** A `null` value removes the path.

```text
commit_files(app_id="b882c472-2a7b-414a-994b-d9c7c3bcdcee",
             message="drop robots.txt again",
             files={"robots.txt": null})
→ {"sha": "c16fdc6adcc7b64b72603a674c05aef8ca708972"}
```

```text
list_files(app_id="b882c472-2a7b-414a-994b-d9c7c3bcdcee")
→ {"ref": "master", "sha": "c16fdc6adcc7b64b72603a674c05aef8ca708972",
   "files": [{"path": "index.html", "kind": "blob", "size": 815},
             {"path": "style.css", "kind": "blob", "size": 563}]}
```

A `null` value is the only way to delete a path through a changeset. That is
why an absent path means "leave the file as it is". The two actions need
different forms, and a map holds one value per path. Then comes the second
deploy, of `c16fdc6a`:

```text
deploy(app_id="b882c472-2a7b-414a-994b-d9c7c3bcdcee",
       channel_id="17a905c3-9365-4c1e-be98-7823f83a2647",
       sha="c16fdc6adcc7b64b72603a674c05aef8ca708972")
→ {"deploy_id": "2acbf6ad-91b6-40ce-9e43-c3d1840ffb3b",
   "channel_id": "17a905c3-9365-4c1e-be98-7823f83a2647",
   "status": "queued"}
```

**No deploy ever carried `robots.txt`.** The file existed in one commit only,
in the repo and nowhere else. The site served commit 1 for that whole period.
The next deploy came after the file was gone. That result is not a fault. It
shows you that a commit is not a deploy. A `git push` with no `deploy` after
it gives the same result.

## Verify it

```text
get_deploy_log(app_id="b882c472-2a7b-414a-994b-d9c7c3bcdcee",
               channel_id="17a905c3-9365-4c1e-be98-7823f83a2647",
               deploy_id="df8750a5-ada3-45fe-b3b1-1543ec14873d")
```

That is the first deploy. The transcript shows short ids. It omits the nginx
entrypoint lines and the routine per-channel lines for the network, the blob
route and the channel row:

```text
[2026-07-31T23:20:42+00:00] deploy begin id=df8750a5-... channel=17a905c3-... sha=0bf2770b...
[2026-07-31T23:20:42+00:00] git_sync sha=0bf2770b0ab04b119bc4138faf2d9b1a270ab095
[2026-07-31T23:20:42+00:00] git_sync ok: synced app=b882c472-... channel=17a905c3-... sha=0bf2770b...
[2026-07-31T23:20:42+00:00] start_static_site template=static
[2026-07-31T23:20:43+00:00] start_static_site ok: container=07e5fbe4b902...
[2026-07-31T23:20:43+00:00] health_check container=07e5fbe4b902... port=80 timeout=10.0s
[2026-07-31T23:20:43+00:00] health_check ok
[2026-07-31T23:20:43+00:00] [container] 2026/07/31 23:20:43 [notice] 1#1: nginx/1.31.3
[2026-07-31T23:20:43+00:00] [container] 2026/07/31 23:20:43 [notice] 1#1: start worker processes
[2026-07-31T23:20:43+00:00] [container] 10.77.1.5 - - [31/Jul/2026:23:20:43 +0000] "GET / HTTP/1.1" 200 815 "-" "Python-urllib/3.13" "-"
[2026-07-31T23:20:43+00:00] caddy ensure_route hostname=recipe-commit-files-docs.xhostd.app upstream=10.77.1.5:32050
[2026-07-31T23:20:43+00:00] caddy ensure_route ok
[2026-07-31T23:20:43+00:00] pinned deployed sha 0bf2770b0ab04b119bc4138faf2d9b1a270ab095
[2026-07-31T23:20:44+00:00] deploy success
```

The last line is the important one: `deploy success`. Poll the log until it
ends with that line, or with a line that names the failure.

**`git_sync` is the first step, and it tells you the main fact.** The deploy
fetches a commit from a git repo, exactly as it does after a push. At that
point nothing separates this deploy from a deploy after a push. The deploy
cannot see how the commit came into the repo.

**The log holds no `[build]` line.** This is the `static` template. It builds
nothing, it gives the committed files to nginx, and the whole log covers two
seconds. `health_check ... port=80 timeout=10.0s` is the static template's
window. nginx already runs, so the probe waits for no process of yours. The
probe itself is the `"GET / HTTP/1.1" 200 815` line, from nginx's own access
log. [Recipe: static site](https://docs.xhostd.com/guides/recipes-static)
reads a static deploy log in full.

The second deploy's log is the same log with two more lines. The first deploy
does not print them, because it replaces no container:

```text
[2026-07-31T23:26:36+00:00] caddy ensure_route ok
[2026-07-31T23:26:36+00:00] stop_and_remove old container=07e5fbe4b902...
[2026-07-31T23:26:36+00:00] stop_and_remove ok
```

`07e5fbe4` is the container from the first deploy. xhostd retires it only
*after* the new container passes its health check, and after the hostname
points at the new container. A failed redeploy thus keeps the previous
version live.

Then the proof, against the live demo:

```bash
$ curl -sS -o /dev/null -w '%{http_code} %{content_type} %{size_download}\n' https://recipe-commit-files-docs.xhostd.app/
200 text/html 815

$ curl -sS -o /dev/null -w '%{http_code} %{content_type} %{size_download}\n' https://recipe-commit-files-docs.xhostd.app/style.css
200 text/css 563
```

815 and 563 are the byte counts of the two files in this guide. Both bodies
are byte-identical to those files. The number 815 is also in the health
probe's access line above, because nginx serves the same file there.

A later `git clone` of `https://git.xhostd.com/docs/recipe-commit-files.git`
shows the three commits above, in order. Its working tree is byte-identical
to the two files here. That is the last fact about `commit_files`: it makes
ordinary commits, on the same repo that a push uses. On a machine with git,
you clone that repo and continue your work.

## When it goes wrong

### The app is connected to GitHub

`commit_files` is refused outright, with

```text
this app is GitHub-connected; xhostd mirrors this repo read-only. Push to GitHub instead — changes sync to xhostd automatically.
```

There is no way around this refusal. On a connected app, GitHub is the
source, and the xhostd repo is a read-only mirror of GitHub. Send the code to
GitHub, and xhostd syncs it from there. This guide describes the message and
does not capture it. The demo app has no GitHub source, and a connection to
GitHub would stay in place. `sync_git` is not the answer either. It refreshes
that mirror, and it does not put code onto an app.

### The commit succeeded but the site did not change

`commit_files` returned a `sha` and did no more. It stores code, but it does
not deploy code. Call `deploy` with that `sha`. A push to git with no deploy
gives the same result, so this failure occurs on both paths. If you do not
know which commit is live, call `get_app` for the channel's `current_sha`.
Call `list_files` for the content of the repo. If the two disagree, the repo
is ahead and a deploy is absent.

### You send the whole tree on every edit

It works. The platform applies every changeset on top of the branch head, so
a file that you send again with no change does no damage. But it costs you
the text of the whole project on every edit. Send only the paths that change.
If git is available on the machine where you work, stop with `commit_files`.
[Recipe: static site](https://docs.xhostd.com/guides/recipes-static) gives
the four steps, and a push sends the diff, not the project.

### The file you need to commit is not text

The values in `files` are strings, and the platform writes them into the repo
as UTF-8. There is no binary form and no base64 alternative. An image, a
font, a favicon or a zip file cannot go through `commit_files`. A static site
with such a file needs git, or a URL to a file on another host. This guide
describes this limit and does not capture it, because the worked example is
two text files. The platform also checks each path. A changeset path must be
relative, and it must contain no `.` or `..` segment.
