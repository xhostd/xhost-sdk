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

**Ownership transfer takes two people.** The owner starts a transfer to an
existing member, and nothing moves until that member accepts. The transfer
stays pending until then: the owner can cancel it, and the recipient can
decline it. A project holds one pending transfer at a time, so cancel the
pending one before you transfer the project to somebody else. The recipient
accepts in the web console, so they need an account with a verified email
address; an account an agent registered has none until someone adds one.

Accepting moves the project onto your account, and its channels count against
your plan from that moment. If your plan has no room for them, the console says
so and the transfer stays pending until you upgrade. The previous owner stays
on the project as an admin.

Each running channel restarts under the new owner's account. The restart is a
blue-green swap, the same one a deploy uses, so the old container keeps serving
until the new one is healthy. The accept is refused while the project has a
deploy in progress; retry once it finishes.

**Accepting can also move the project's databases.** When you keep your own
databases on another database server of the same machine, accepting moves the
project's databases into your account. The console says so before you accept.
Each app pauses for about 15 seconds while its database moves. Until the move
ends, you cannot create or delete projects or channels, and deploys and git
pushes wait during each pause. The accept is refused while one of
the project's databases is not ready yet, or while an export of the
project runs; retry once it finishes.

**A channel URL keeps the previous owner's username.** The platform does not
rename a channel hostname, so a transferred project serves from the same
address it always did. While that holds, the previous owner cannot create a new
project under the transferred name, because that name would derive the same
hostname.

All four steps are console-only. No agent-permission setting opens them, so no
agent can hand your project away, or take one on for you.

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
