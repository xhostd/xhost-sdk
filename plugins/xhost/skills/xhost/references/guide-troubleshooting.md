# Troubleshooting

Start with the exact project, channel, and time of the problem. Separate a
failed deployment attempt from the revision that is currently serving traffic.

## A deployment failed

Open the project's Activity page in the
[console](https://console.xhostd.com/projects). Select the deployment and read
its build log. Check dependency installation, the launch command, and readiness
before retrying. Preserve the error text without exposing credentials.

A git push alone does not deploy. A queued response is not success. Follow the
[deployment procedure](https://docs.xhostd.com/guides/deployments) through its
terminal state.

## The client blocked the action

Distinguish a client approval prompt from a platform permission error. The
[blocked-deploy guide](https://docs.xhostd.com/guides/client-blocked-deploy)
explains how to identify the source. Do not enable unrelated protected actions
as a workaround.

## The app is slow or unavailable

Check the serving channel, recent changes, and account resource measurements.
The [slow-app guide](https://docs.xhostd.com/guides/diagnose-slowness) walks
through evidence collection. A deployment log explains a build; runtime
metadata describes the process; traffic measurements describe requests.
None substitutes for the others.

Do not assume a missing measurement is zero usage or a failed latest attempt
means the serving app is down. Shared-project members may legitimately lack
the owner's aggregate resource data.

## Data or files are missing

Check the target channel and which store the application uses. Review available
recovery points before attempting a restore. See
[Data and recovery](https://docs.xhostd.com/guides/data-and-recovery) for the
distinction between code, database, and file recovery.

## Escalate with useful context

Include the project owner/name, channel, deployment identifier if relevant,
time and timezone, expected result, actual result, and sanitized error text.
Do not include tokens, environment secrets, or customer data. Use console
feedback or [contact support](mailto:support@xhostd.com).
