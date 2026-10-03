# When your client blocks a deploy

You asked for a deploy, and no deploy ran. Your agent client refused the
`deploy` tool call on your machine, before the call left it. This guide tells
you how to recognize that refusal, how to clear it for one deploy, and how to
configure your client so it does not come back.

This guide is written for you, the person at the keyboard. Every step and
every file here is yours to act on. Your agent does not run these steps and
does not edit these files, and it does not need to.

## What happened

Your client runs an automatic permission mode. In that mode a classifier reads
each tool call and decides whether to run it without asking you first. The
classifier treats a production deploy as a call that needs your word, so it
stopped the call and reported the block to your agent.

The call never reached xhostd. xhostd queued no deploy, wrote no log line, and
returned no error, so there is nothing to retry against the API and nothing to
read in a deploy log. This is also not xhostd refusing the deploy. An xhostd
refusal comes back as the reply to the tool call, and it carries an error
message that tells you what to change.

No change on the xhostd side moves this decision. The classifier reads the
call, not the metadata a tool ships with, and the `deploy` tool already
declares the least alarming annotations the protocol allows. The decision
belongs to your client, so the settings that change it are your client's
settings.

## How to recognize it

The message comes from your client itself, not from a tool result. It names
the automatic permission mode's classifier. Your client's auto mode
configuration reference gives the reason it shows as the fixed text
`Blocked by classifier`. A refusal from xhostd looks different: it arrives as
the reply to the tool call, and it carries an error message. An error message
in the tool's reply means the call reached xhostd, and this page does not
apply.

## Restate the intent for this deploy

The quickest remedy is your next message. Name the app and the channel you
want deployed:

```
Deploy my-site to prod.
```

A specific instruction that names the target clears this block. A general one,
such as "ship this", does not, because it leaves the classifier the same
question it stopped on. This remedy covers the deploy in front of you. Your
next session starts from the same place.

## Retry the denied call

Your client keeps the calls it denied. Its auto mode configuration reference
gives these steps for running the one it stopped:

1. Run `/permissions`.
2. Open the **Recently denied** tab.
3. Select the denied `deploy` call.
4. Press `r`.

The call then runs with your approval behind it.

## Allow xhostd deploys in your user settings

A classifier that stops an unattended production deploy is doing its job. The
settings here do not switch that check off. They tell the classifier what your
setup is, so that its judgment about your xhostd deploys is an informed one
and you stay the person who authorized them.

Two keys in `~/.claude/settings.json` carry that description:

- `autoMode.environment` — what your working environment is.
- `autoMode.allow` — prose rules naming what you accept without a prompt.

Set either key, or both. This file is yours to edit.

Two traps make a correct-looking edit do nothing:

- **The classifier reads `autoMode` from your user settings only.** It does
  not read `autoMode` from a project's `.claude/settings.json` or
  `.claude/settings.local.json`. A rule you write in a project file is
  ignored, and nothing reports that it was.
- **A list you set replaces the built-in rules for that key.** Keep the
  literal `"$defaults"` string as an entry in any list you set. Without it,
  you discard the rules your client ships with for that key.

This `autoMode.allow` rule describes an xhostd deploy. If
`~/.claude/settings.json` already holds other keys, merge the `autoMode` key
into the object it already has:

```json
{
  "autoMode": {
    "allow": [
      "$defaults",
      "Deploying an xhostd app through the xhostd MCP tools is routine work: a deploy ships a commit that is already pushed to the app's own repository, and the previous version stays redeployable with the rewind tool."
    ]
  }
}
```

Write the rule in your own words if that description does not match how you
work. The rule is prose, and the classifier reads it as prose.

## The Bash rule in the denial message does not apply

The tail of the denial suggests adding a Bash permission rule. That suggestion
fits a denied shell command. `deploy` is an MCP tool call, and it runs no shell
command, so a Bash rule changes nothing about this block.

## Where these rules come from

The automatic permission mode belongs to your client, not to xhostd. Its
behavior can change at any time, and no xhostd test holds it to what this page
says. This page records what Claude Code did on 2026-09-06. For the current
key names, their defaults, and how they combine, read
[the Claude Code auto mode configuration reference](https://code.claude.com/docs/en/auto-mode-config).

Three claims on this page are xhostd's own, and they hold: the blocked call
never reached xhostd, the `deploy` tool's annotations are already the least
alarming the protocol allows, and the Bash rule in the denial tail does not
apply to an MCP tool.
