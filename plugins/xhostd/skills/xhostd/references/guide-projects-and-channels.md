# Projects and channels

A project owns source code and one or more channels. A channel is a deployment
environment, such as production or a preview, with its own live address.

## Understand the scope

Every project starts with a `prod` channel. Each additional channel uses another
slot in the owner's account-wide channel allowance. A preview is not a separate
billing plan. CPU, memory, database usage, and file storage are account-scoped
resources; adding projects does not multiply that budget.

Code-running channels have their own Postgres database. Static channels do not.
Use the platform-provided environment values instead of deriving credentials or
hostnames yourself. Platform sites use `xhostd.com`; tenant app addresses use
`xhostd.app`.

## Create and inspect a project

Connect an agent using the [setup guide](https://docs.xhostd.com/getting-started).
Ask it to create a project with the template your workload needs. Choose a
[working recipe](https://docs.xhostd.com/guides) before writing the build and
launch scripts.

Open [Projects in the console](https://console.xhostd.com/projects). A project's
Overview separates the currently serving channels from the latest deployment
attempt. A failed attempt does not by itself mean the previous version stopped.

## Use a preview channel

Ask your agent to create a separate preview channel and deploy a revision to it.
Review the returned URL before deploying that revision to production. Verify
the channel's environment values and data setup; do not assume a preview holds
a current copy of production data.

See the [channel API](https://docs.xhostd.com/api#grp-channels) for the exact
creation and deletion contracts. A new channel and a new code deployment are
different operations.

## Work on a shared project

Check the owner shown next to the project name. The API and MCP tools accept
owner-qualified names where documented; the console keeps that owner context
in links. Membership is not ownership. A member must not interpret an unavailable
owner-wide resource summary as zero usage.

Manage invitations and roles in the project's Access section. See
[Access and security](https://docs.xhostd.com/guides/access-and-security).

## Next steps

- [Push source with git](https://docs.xhostd.com/guides/git).
- [Deploy and inspect a change](https://docs.xhostd.com/guides/deployments).
- [Understand recovery](https://docs.xhostd.com/guides/data-and-recovery).
