# Power Automate Guardian

**Status: complete (6 reference files).**

Companion skill to [`power-apps-guardian`](../power-apps-guardian), built
the same way: research-first, category by category, from real production
issues and cross-checked documentation rather than AI training-data memory.

## What's covered

| File | Covers |
|---|---|
| `error-handling-and-retry.md` | Try-Catch-Finally with Scopes, Configure Run After mechanics, retry policy limits and idempotency judgment, the Terminate action |
| `connector-file-locking.md` | Excel/Word connector session-lock races against SharePoint file operations — built from a real production incident |
| `trigger-and-action-limits.md` | Default row/item limits by connector, the pagination-threshold rounding confusion, source-side OData filtering, and self-triggering infinite loops |
| `concurrency-and-parallelism.md` | Trigger-level vs. Apply to Each concurrency, race conditions from shared variables and order dependency, duplicate-record/conflicting-write risks, throttling amplification |
| `alm-and-connections.md` | Run-as identity by trigger type, connection reference consolidation, orphaned/disabled-owner flows, service principal ownership, sharing/co-owner nuances, environment variable deployment gaps |
| `throttling-and-flow-limits.md` | Platform vs. connector-level throttling, daily action-limit tiers, auto-disable conditions, structural per-flow limits, Do Until defaults, and nested-loop action-count multiplication |

**Not covered (intentionally out of scope for now):** Power Automate
Desktop / RPA flows, Copilot Studio / agent flows, Business Process Flows.
Add these as new categories, following the same research-first process,
if they become a real need.

## The connector-file-locking.md story

This file exists because of a real incident: a flow reading an uploaded
Excel file's table data, then immediately trying to move the file to a
processed/error folder, failed intermittently with an `SPFileLockException`.
The cause — the Excel connector holds a co-authoring-style session open for
minutes after the read action completes — isn't obvious from the error
message and isn't something an AI would know without being told. The fix
(copy the file to staging, read the copy, never let the risky connector
touch the file that needs to be moved) is now documented as the default
pattern this skill should reach for, verified against Microsoft's own
connector documentation and independent practitioner sources.

This is the model for how this skill should keep growing: real issues,
verified against multiple sources, written up with enough mechanism detail
that the "why" transfers to adjacent situations (Word documents hit the
same issue; PowerPoint plausibly does too).

## How the files connect to each other

Several files deliberately cross-reference each other rather than
duplicating content — worth knowing when reading or extending them:
- `throttling-and-flow-limits.md` explains that an owner leaving the
  organization (`alm-and-connections.md` #3) silently drops a flow's
  paginated-items ceiling from 100,000 to 5,000, since ownership loss
  triggers a performance-profile downgrade.
- `concurrency-and-parallelism.md`'s throttling-amplification note and
  `throttling-and-flow-limits.md`'s connector-level throttling section
  cover the same underlying risk from two angles (parallelism causing
  throttling vs. understanding throttling once it happens).
- `connector-file-locking.md`'s retry guidance for the copy/delete staging
  steps points back to `error-handling-and-retry.md`'s general retry
  policy guidance rather than repeating it.

## Shared verification protocol

This skill follows the same verification discipline as `power-apps-guardian`
— see [`../power-apps-guardian/references/verification-protocol.md`](../power-apps-guardian/references/verification-protocol.md)
rather than duplicating it. Connector lock durations, API quotas, action-
limit tiers, and trigger-concurrency defaults in particular are flagged
Medium/Medium-high volatility — verify current figures live before
treating cited numbers as permanent.

## Contributing

Same approach as `power-apps-guardian`: if you hit a real issue this skill
doesn't catch, add it using the same schema (pattern → why it breaks → fix
→ scope → severity → live-search trigger), backed by actual research, not
assumption.
