---
name: power-automate-guardian
description: Helps write Power Automate cloud flows that avoid failures that only show up in production — silent error handling gaps, connector file-locking races (Excel/Word/SharePoint), trigger/action data limits, self-triggering infinite loops, concurrency/race condition bugs, ALM/ownership issues (run-as identity, orphaned flows, service principals, environment variable deployment gaps), and throttling/flow-limit issues (platform vs. connector-level throttling, auto-disable conditions, Do Until defaults, nested-loop action multiplication). Use whenever writing or reviewing a Power Automate flow — file operations, bulk data retrieval, "when created or modified" triggers, Apply to Each loops, deployment/ownership decisions, or high-volume/nested-loop processing. Checks work against reference files before finalizing, follows the same verification protocol as the companion power-apps-guardian skill, and documents what was caught for future reference.
---

# Power Automate Guardian

**Status: complete (6 reference files).** Built the same way as the
companion [`power-apps-guardian`](../power-apps-guardian) skill:
research-first, category by category, schema-consistent reference files,
including one built directly from a real production incident
(`connector-file-locking.md`). This isn't a claim of universal coverage —
see "Known gaps" below for what's intentionally out of scope.

## Read this first, every time: the verification protocol

This skill shares its verification discipline with `power-apps-guardian`
— see [`../power-apps-guardian/references/verification-protocol.md`](../power-apps-guardian/references/verification-protocol.md).
In short: never state a specific number, quota, or duration with
confidence without either verifying it live or flagging it as potentially
stale. Several categories here (connector lock durations, connector page
sizes, API quotas, licensing, trigger-concurrency defaults, service
principal licensing, solution-ownership tooling, action-limit tier
figures) overlap directly with things the Power Apps skill already flags
as high-volatility — treat that overlap as reinforcement, not duplication
to ignore.

## What's covered

1. `references/error-handling-and-retry.md` — Try-Catch-Finally with
   Scopes and Configure Run After, retry policy configuration and limits,
   idempotency/re-run safety, when NOT to retry (create/send/charge-money
   actions), the Terminate action for explicit failure status.
2. `references/connector-file-locking.md` — built from a real production
   incident (Excel Online connector holding a file lock after "List rows
   present in a table," causing intermittent failures on a subsequent Move
   file step). Covers documented lock durations by connector variant, the
   copy-to-staging fix pattern, alternative workarounds, and a documented
   false-positive case (identical error message, unrelated schema-mismatch
   root cause).
3. `references/trigger-and-action-limits.md` — default row/item limits by
   connector (SharePoint 100, Excel 256, Dataverse 5,000, Outlook 25 with
   no pagination), the "threshold rounds up to page size" confusion,
   source-side OData filtering, and self-triggering infinite loops from
   "when created or modified" triggers combined with same-record write-back.
4. `references/concurrency-and-parallelism.md` — the two distinct
   concurrency settings (trigger-level vs. Apply to Each), race conditions
   from shared variable manipulation, order-dependent loops, duplicate-
   record/conflicting-write races, and throttling amplification from
   aggressive parallelism.
5. `references/alm-and-connections.md` — the flow-specific counterpart to
   the Power Apps skill's ALM file: run-as identity by trigger type,
   connection reference consolidation, orphaned/disabled-owner flows,
   service principal ownership, sharing/co-owner permission nuances,
   environment variable value-deployment gaps, and solution-ownership
   transfer limitations.
6. `references/throttling-and-flow-limits.md` — the two distinct
   throttling layers (platform license-tier limits vs. connector-specific
   service protection), the 6,000/40,000/250,000 daily action tiers and
   what counts as an action, auto-disable conditions, structural limits
   (500 actions/8 nesting levels per flow), Do Until's non-intuitive
   defaults, and the nested-loop action-count multiplication effect —
   including a direct link back to `alm-and-connections.md` (an owner
   leaving drops a flow to the Low performance profile, silently cutting
   its pagination ceiling from 100,000 to 5,000).

## Known gaps
This skill does not cover: Power Automate Desktop / RPA-specific flows,
approvals-connector-specific patterns beyond what's covered incidentally,
Copilot Studio / agent flows, or Business Process Flows. These weren't on
the original priority list — add them as new categories, following the
same research-first process, if they become a real need.

## How to use this skill

1. **Any flow with more than a couple of actions touching an external
   system** → check `error-handling-and-retry.md` before considering it
   finished. Default to a Try-Catch-Finally Scope structure rather than
   scattered per-action Run-After branches, and evaluate retry policy per
   action based on idempotency — don't leave every action on default
   retry without thinking about whether it's safe.
2. **Any flow reading from or writing to an Excel or Word file, followed
   by a file-management operation (Move, Delete, Copy-overwrite, Update
   properties) on that same file** → check `connector-file-locking.md`
   *before* finalizing the flow's structure. Default to the copy-to-
   staging pattern (never let the Excel/Word connector touch the file
   that needs to be moved/deleted) rather than a fixed delay or blind
   retry loop.
3. **Any "file is locked" or "unable to acquire a lock" error on an Excel
   table write action** → check `connector-file-locking.md` #4 before
   assuming it's a session lock — rule out schema mismatches and
   file-reference-by-Id-vs-Path issues first.
4. **Any flow with a "Get items"/"Get rows"/"Get files" action, or a
   "when created or modified" style trigger** → check
   `trigger-and-action-limits.md`. Verify the connector's default page
   size isn't silently truncating results, and — critically — check
   whether the flow writes back to the same record it was triggered by,
   which risks a self-triggering infinite loop unless a trigger condition
   is added.
5. **Any Apply to Each loop, or any flow whose trigger could fire multiple
   times in close succession for related data** → check
   `concurrency-and-parallelism.md`. Confirm loop iterations are
   independent (no shared variables, no order dependency) before enabling
   Degree of Parallelism, and confirm retry policy is in place before
   raising parallelism enough to risk throttling.
6. **Any flow being deployed across environments, shared with co-owners,
   or owned by an individual rather than a service principal** → check
   `alm-and-connections.md`. Identify trigger type before diagnosing
   inconsistent behavior across users, verify environment variable
   *values* (not just definitions) after deployment, and confirm
   business-critical flows have multiple owners or service-principal
   ownership rather than a single point of failure.
7. **Any flow processing meaningful volume, using nested loops, using Do
   Until, or expected to run at production scale/frequency** → check
   `throttling-and-flow-limits.md`. Estimate actual daily action
   consumption against the owner's license tier (accounting for loop
   multiplication), distinguish platform-level from connector-level
   throttling when diagnosing 429s, and check auto-disable risk for
   infrequently-run or error-prone flows.
8. **At the end of the working session, produce a short changelog** (see
   below), matching the format used by the Power Apps skill so both can
   be reviewed together for a project touching both canvas apps and flows.

## End-of-session documentation format

```
### Power Automate Guardian — session notes

| Issue | Connector/action | Root cause | Fix applied | Reusable snippet? |
|---|---|---|---|---|
```

Keep entries terse — one row per distinct issue caught, not per action
touched.

## Reference files

- `references/error-handling-and-retry.md` — read before finalizing any
  flow with meaningful external-system logic.
- `references/connector-file-locking.md` — read whenever a flow's
  structure has an Excel/Word connector action followed by a file-
  management operation on the same file.
- `references/trigger-and-action-limits.md` — read whenever a flow
  retrieves a list/table/library, or uses a "when created or modified"
  style trigger.
- `references/concurrency-and-parallelism.md` — read whenever a flow uses
  Apply to Each with more than a trivial number of items, or has a trigger
  that could fire multiple times in close succession for related data.
- `references/alm-and-connections.md` — read whenever a flow is deployed
  across environments, shared with co-owners, or ownership/identity
  behavior needs diagnosing.
- `references/throttling-and-flow-limits.md` — read whenever a flow
  processes meaningful volume, uses nested loops or Do Until, or is
  expected to run at production scale/frequency.
