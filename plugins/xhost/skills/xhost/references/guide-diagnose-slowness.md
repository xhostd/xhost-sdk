# Diagnose a slow app

An app is slow, or it answers with an error, or the deploy log is clean but
the URL is not correct. One tool answers all three questions. Call
`get_app_health(app_name, channel)`. It reads the resources of the account,
the container of the channel, the recent builds and the channel database in
one call. It then gives you a diagnosis and, with each cause, the action for
that cause.

This guide is for an agent that acts for a user. It shows the order to read
the reply, and the rule that each part of the reply carries.

## Call the health tool first

```
get_app_health(app_name="my-site", channel="prod")
```

Both arguments are necessary. The diagnosis is per channel, so there is no
all-channel form. If two apps that you can reach have the same name, qualify
the name as `owner_username/app_name`.

There is no window argument. The tool reads the last hour of resource
figures, and the last day of build events.

The reply has this shape:

```
{
  "app_id": "...",
  "channel_id": "...",
  "resource": { "available": true, "reason": null, ... },
  "runtime":  { "available": true, "reason": null, ... },
  "build":    { "available": true, "reason": null, ... },
  "database": { "available": true, "reason": null, ... },
  "findings": [ ... ]
}
```

## Read `findings` first

`findings` is the diagnosis. The four blocks are the measurements behind it.
Read the findings, and act on them. Do not read the figures first, and do not
compute your own conclusion from them.

Each finding carries five fields:

| Field | What it holds |
|---|---|
| `code` | A stable name for the cause, such as `build_oom_killed` |
| `severity` | `info`, `warning` or `critical` |
| `what` | The fact, in plain words, for the user |
| `why` | The cause, in plain words, for the user |
| `action` | The rule: what to do, and who does it |

**Never parse `what` or `why` to decide what to do.** They are text for a
person. `action` is the machine-readable rule, and it is the only field that
decides your next step.

`findings` is never empty. A channel with no fault carries one finding with
the code `healthy` and the action `none`.

## Obey `action`

`action.do` is a closed set of six verbs:

| `do` | What you do |
|---|---|
| `wait` | The cause can clear on its own. Read the health again after `retry_after_seconds`. |
| `retry` | Make the same call again after `retry_after_seconds`. |
| `change_code` | The app or its query needs an edit. A plain retry fails again. |
| `upgrade_plan` | The account is at a limit of its plan. Only a larger plan raises that limit. |
| `contact_support` | The xhost team owns the cause. |
| `none` | There is nothing to do. |

`action.retry_after_seconds` carries a number for `retry` and for `wait`
alone. It is null for every other verb. The key is always there, so read the
value and not the key. Thus you never guess a delay, and you never apply a
delay where none belongs.

`action.actor` names who acts. `agent` is you. `user` is the person you work
for. **Do not try to satisfy a `user` action yourself.** Only the user can pay
for a larger plan, and only a person can speak to the xhost team. Relay the
`what` and the `why` text to the user, and stop there.

A reply can carry more than one finding. Act on the `critical` findings
first.

## Then read the blocks

The blocks hold the figures behind the findings. Read them when you need the
detail, or when you write a report for the user.

- **`resource`** — recent CPU, memory, memory pressure, disk pressure and
  OOM-kill counts of the account.
- **`runtime`** — the container of this channel: whether it runs, its exit
  code, whether the kernel stopped it, and its restart count.
- **`build`** — how many recent builds of this channel the kernel stopped for
  memory, and when the last one was.
- **`database`** — the size of the channel database, its cache-hit percent,
  and its slowest statements.

### An unavailable block is not an error

Every block carries `available` and `reason`. When `available` is false, the
platform could not read that source, and `reason` states the cause in plain
words. The call still succeeds. Read `reason`, and tell the user what it
says.

A figure the platform could not read is null. It is never a zero. A zero in
that place would be a false measurement.

Three states are normal, and none of them is a fault:

- **`resource` is unavailable to a member of a shared app.** Those figures
  cover the whole account of the app owner, over every app that the account
  runs. They are not per channel, so a member does not read them.
- **`database.top_statements` carries its OWN `available`.** An unavailable
  `top_statements` beside an available `database` is normal. The common
  cause is a database server with no `pg_stat_statements` extension, but a
  failed read gives the same shape. Read `reason` for the true cause. The
  size and the cache-hit figures are still correct.
- **`runtime` is unavailable before the first deploy.** A channel with no
  container yet has no status to report.

On a statement, `truncated` tells you that the query **text** is cut. It does
not tell you that the statement stopped early.

## Where to go after the diagnosis

The health tool names the cause. Three other tools give you the detail.

- **`get_runtime_log`** — on the `container_exited_nonzero` finding, or on
  `container_restart_loop`, read the error text that the app itself wrote.
  Call it with no `command` for the status header, then with
  `command="tail -n 200 app.log"` for the lines. The
  [Best practices](https://docs.xhostd.com/guides/bkm) guide covers this tool
  in full.
- **`get_deploy_log`** — on a `build_oom_killed` finding, read the build that
  the kernel stopped, and find the step that used the memory.
- **`get_app_stats`** — the health tool reads one hour. Use the stats tool
  for a 24-hour, 7-day or 30-day view of the traffic.

## A worked example

A channel answers slowly. The health call gives two findings:

```
"findings": [
  {
    "code": "db_slow_statements",
    "severity": "warning",
    "what": "At least one statement of this database is slow.",
    "why": "The statement holds the request open until it ends. The top-statement list names it.",
    "action": { "do": "change_code", "actor": "agent", "retry_after_seconds": null }
  },
  {
    "code": "memory_pressure_elevated",
    "severity": "warning",
    "what": "The account waits for memory more than usual.",
    "why": "The memory pressure is above the normal range but below the high threshold. It can fall again on its own.",
    "action": { "do": "wait", "actor": "agent", "retry_after_seconds": 300 }
  }
]
```

Read the actions, not the figures. The first action is `change_code` with the
actor `agent`, so you act. Open `database.top_statements`. Find the statement
with the high `mean_exec_ms`. Add the index that it needs. Then deploy the
app again.

The second action is `wait` with a delay of 300 seconds, so you change
nothing for it. Call `get_app_health` again after that delay. Then see
whether the pressure fell.
