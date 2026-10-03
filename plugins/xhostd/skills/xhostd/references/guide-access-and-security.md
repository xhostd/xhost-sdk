# Access and security

Connecting a tool, sharing a project, and granting a sensitive capability are
different decisions. Choose the smallest scope that supports your task.

## Connect your account

Follow the [tool setup](https://docs.xhostd.com/getting-started) instructions.
Account sign-in authenticates you to xhostd; it is not the same as adding Google
login to your own application.

Static documentation does not read your account identity. The Console link
opens your dashboard or sends you through sign-in.

## Share a project

Use the project's Access page to manage members and invitations. Check the
owner and project name before changing roles. Being a member does not make
you the owner or grant access to the owner's other projects.

Ownership transfer and project deletion remain in project Settings, with their
existing review and confirmation requirements.

## Manage agent permissions

The account has a default protected-action setting; a project can inherit or
override it. Connecting an agent does not silently enable sensitive actions.
Read the project's effective setting before asking an agent to reveal secrets
or open external access.

The human console owns this switch. An agent cannot grant itself the protected
capability. If a client blocks an operation before it reaches the platform, see
[Client blocks a deploy](https://docs.xhostd.com/guides/client-blocked-deploy).

## Keep credentials separate

Account access keys and SSH keys have their own console pages. Project members,
invitations, and external connection settings belong to project Access.
Environment values belong to Environment. Secret fields are not an activity
log or an agent handoff payload.

Treat revealed and newly issued credentials as secrets. Do not place them in
repository source, screenshots, documentation search, or feedback messages.

## Publish access deliberately

Follow the [Google sign-in guide](https://docs.xhostd.com/oauth) to protect your
application, and the [custom domain guide](https://docs.xhostd.com/domains) to
attach a hostname. Review external Postgres or raw-TCP access independently of
the app's HTTPS endpoint; they are not equivalent exposures.

Read the [API reference](https://docs.xhostd.com/api) and
[MCP versus console matrix](https://docs.xhostd.com/mcp-tools#mcp-vs-console)
for the current capability boundaries.
