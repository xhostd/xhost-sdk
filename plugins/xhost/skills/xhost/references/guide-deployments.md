# Deployments

Deploy a specific revision to a specific channel, then inspect the outcome.
Sending source to the repository and making it live are separate operations.

## Push, then deploy

Use [git](https://docs.xhostd.com/guides/git) to send code to the project's
repository. A push stores the commit; it does not deploy it. Use `commit_files`
only when git is unavailable, as described in
[Ship without git](https://docs.xhostd.com/guides/recipes-commit-files).

Call the [deploy tool](https://docs.xhostd.com/mcp-tools#deploy) with the app,
channel, and commit. Prefer a full SHA when you need the same immutable target
across review and execution. A branch reference resolves at deploy time.

## Follow the operation

The deployment call returns queued work, not proof of success. Read
[get_deploy_log](https://docs.xhostd.com/mcp-tools#get_deploy_log) and inspect its
status until it is terminal. Build output and runtime output answer different
questions. A missing build log does not establish that the running app is healthy.

The project's Activity page in the [console](https://console.xhostd.com/projects)
separates deployment history, build-log windows, runtime metadata, and audit
events. Use Refresh to read additional build output. The console does not run
arbitrary shell commands or present runtime metadata as a runtime log stream.

## Deploy an existing revision from the console

Open the project Overview and choose its deployment action. Select the channel
and enter the full 40-character commit SHA. Review the project, owner, channel,
and current serving revision before confirming.

The review expires after ten minutes and is tied to your session and target.
If the serving revision changes, review again. Permission is checked again at
execution; an old review does not retain access you no longer have.

## Recover code without confusing it with data

The [rewind tool](https://docs.xhostd.com/mcp-tools#rewind) uses a retained
previous successful image where supported. Deploying an older SHA rebuilds that
revision instead. Neither operation is a database or file restore.

> Check the application's schema expectations before running older code against
> current data. A deployment review does not freeze environment values or undo
> data changes.

For database and file recovery, use
[Data and recovery](https://docs.xhostd.com/guides/data-and-recovery). For failures,
start with [Troubleshooting](https://docs.xhostd.com/guides/troubleshooting).
