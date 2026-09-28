# Power Automate Error Handling & Retry Reference

Power Automate has no native try/catch syntax — this file covers how to
build the equivalent using Scopes and Configure Run After, and how to use
built-in retry policies correctly rather than either ignoring failures or
retrying things that shouldn't be retried.

Cross-reference: `connector-file-locking.md` is a specific, real-world
instance of the retry/root-cause-vs-symptom judgment this file covers more
generally — read both together when a flow involves file operations.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written without this knowledge |
| Why it matters | The real failure mode this prevents |
| Correct pattern | The fix, with real structure |
| Scope | Where this applies |
| Severity trigger | When this stops being optional |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. No error handling at all — the default AI-generated flow shape
- **Default habit:** A flat sequence of actions with default "is
  successful" run-after on everything. The first action that fails stops
  the entire flow, with no notification, no logging, no cleanup, and often
  no clear indication to anyone that it happened until a person notices
  something downstream didn't happen.
- **Why it matters:** This is the Power Automate equivalent of an unwrapped
  Patch in Power Apps (cross-reference the Power Apps skill's
  `error-handling.md` #1) — it's the single most common and costly gap in
  AI-generated automation, because a flow that "looks like it works" in
  testing gives no signal that it's fragile until it fails in production,
  often silently.
- **Correct pattern:** Every flow with more than a couple of actions
  touching an external system (SharePoint, Excel, an HTTP call, a database)
  should have a deliberate Try-Catch-Finally structure (#2) and appropriate
  retry policies (#3) — not bolted on after a failure, but designed in
  from the start.
- **Scope:** Universal — applies to essentially every production flow.
- **Severity trigger:** Any flow with more than one or two actions, or any
  flow whose failure would be costly/hard to notice.
- **Live-search trigger:** Low.

## 2. Try-Catch-Finally with Scopes and Configure Run After
- **Default habit:** Scattering individual "Configure run after" branches
  action-by-action instead of grouping logic, producing a tangle of
  branches that's hard to read or maintain — or not using Configure Run
  After at all.
- **Why it matters:** `Configure Run After` only evaluates the
  **immediately preceding action** — if Action A → B → C, and C is
  configured to run after B "has failed," that check only covers B, not A.
  Individual action-by-action branching therefore doesn't reliably catch
  failures from earlier in a chain unless every single link is configured
  correctly, which becomes unmaintainable at any real flow size. **Scopes
  solve this** by grouping a block of actions so a single Catch scope can
  respond to a failure anywhere inside the Try scope, not just the last
  action in it.
- **Correct pattern — the standard structure**, confirmed consistently
  across multiple independent sources as "the" recommended approach:
  1. **Try scope** — contains the main business logic (API calls,
     Dataverse/SharePoint operations, calculations).
  2. **Catch scope** — Configure Run After set to trigger when the Try
     scope **has failed**, **is skipped**, or **has timed out** (not just
     "has failed" alone — a skipped or timed-out Try scope is still a
     failure state that needs handling). Contains error-handling logic:
     log the error, notify, attempt recovery, or roll back.
  3. **Finally scope** — Configure Run After set to run regardless of
     outcome (after Try/Catch's succeeded, failed, skipped, *and* timed-out
     states). Contains cleanup that must always happen: closing sessions,
     updating a status field, final logging.
  4. The four available Run-After states on any action/scope: **is
     successful** (default), **has failed**, **is skipped**, **has timed
     out**.
- **Making the Catch block actually actionable, not just noise:** include
  the failed action's name, the error message/status code, and a link to
  the specific failed run (built via the `workflow()` function) in
  whatever notification the Catch scope sends — a bare "something failed"
  message sent to a channel nobody monitors accomplishes nothing.
- **Per-item granularity inside loops:** for a flow processing multiple
  items (`Apply to each`), placing a Try/Catch scope pair **inside** the
  loop (rather than around the whole loop) gives per-item error detail and
  allows one item's failure to be logged/handled without necessarily
  stopping processing of the remaining items — cross-reference #5's
  concurrency notes, since this interacts with how Apply to Each runs.
- **Scope:** Universal — the standard structure for any flow with
  meaningful external dependencies.
- **Severity trigger:** Any flow calling more than one external
  connector/service, or any flow where a partial failure needs to be
  distinguished from complete success.
- **Live-search trigger:** Low — this pattern is stable and well-
  established across the ecosystem.

## 3. Retry policies — knowing the actual configuration limits and when
   NOT to retry
- **Default habit:** Leaving retry policy at its default for every action
  without considering whether the action is safe to retry, or manually
  building fragile Do-Until retry loops when a built-in retry policy would
  do the job more simply.
- **The actual configuration space** (per action, in Settings → Retry
  Policy):
  - **Default** — a built-in policy, commonly around 4 attempts with
    exponential backoff for many connectors (verify per-connector, as this
    can vary).
  - **None** — disables automatic retry entirely.
  - **Fixed Interval** — retry N times with a constant delay between
    attempts.
  - **Exponential Interval** — retry with increasing delay between
    attempts, generally the right choice for calls to external APIs/
    services that may be transiently overloaded.
  - **Configuration limits (platform-wide):** maximum of **90 retries**,
    minimum interval of **5 seconds**, maximum delay of **1 day**.
- **The critical judgment call: not everything should be retried.**
  Read operations and genuinely idempotent updates (an update that
  produces the same end state no matter how many times it's applied) are
  safe to retry automatically. Actions that **create records, send
  messages/emails, charge money, or make changes that are hard to undo**
  need much more caution — a naive retry on a failed "Create item" or
  "Send an email" action can silently create duplicates or send the same
  notification multiple times if the failure was actually a timeout on the
  *response* rather than a true failure of the *operation itself* (the
  create/send may have actually succeeded server-side even though the
  flow saw a failure).
- **The throttling trap — retrying can make things worse, not better:**
  every action execution counts toward a connector's request quota
  **regardless of whether it succeeds or fails**, and retry attempts and
  pagination requests count too (cross-reference the Power Apps skill's
  `licensing-and-limits.md` — API quotas are shared platform-wide, not
  siloed per product). Tuning toward *more* aggressive retries in response
  to a 429 (throttled) response can actually worsen the throttling by
  generating more requests against an already-strained quota. For 429
  responses specifically, favor exponential backoff (increasing delay)
  over fixed-interval or aggressive retry counts.
- **Scope:** Universal, with the create/send/charge-money caveat being the
  key judgment point.
- **Severity trigger:** Any action that isn't a pure read — evaluate
  idempotency explicitly before setting retry policy, don't leave it on a
  default without thinking about it.
- **Live-search trigger:** Medium — the specific default retry attempt
  count and backoff behavior can vary by connector and has been adjusted
  by Microsoft before; verify current specifics if precise retry timing
  matters for a design.

## 4. Ending a flow explicitly as failed when it should be
- **Default habit:** A flow whose business logic determines something has
  gone wrong (e.g. a validation check fails, or a Catch scope's recovery
  attempt also fails) simply completes "successfully" from Power
  Automate's own run-history perspective, because no action actually
  errored — it just did nothing useful.
- **Why it matters:** A flow run showing green/"Succeeded" in run history
  when it actually failed to accomplish its purpose defeats monitoring and
  alerting built around run status.
- **Correct pattern:** Use the **Terminate** action with status set to
  **Failed** (and a custom error message/code) at the point a flow
  determines it cannot complete its actual purpose, even if no connector
  action technically errored. This makes run history, and any monitoring
  built on top of it, accurately reflect reality.
- **Scope:** Any flow with business-logic-level failure conditions that
  aren't the same as a connector action throwing an error.
- **Severity trigger:** Any flow with validation logic or conditional
  success criteria beyond "did every action run without erroring."
- **Live-search trigger:** Low.

## 5. Idempotency and re-run safety
- **Default habit:** Designing a flow without considering what happens if
  it's re-run (manually resubmitted after a failure, or accidentally
  triggered twice for the same input).
- **Why it matters:** Cross-reference #3's create/send/charge-money
  caution — a flow that isn't idempotent can produce duplicate records,
  duplicate notifications, or double-charged/double-processed side effects
  if re-run, which is often exactly what happens when someone manually
  resubmits a failed run to "just get it done."
- **Correct pattern:** Where feasible, design create/write operations to
  check for existing state first (e.g., check if a record with this
  identifier already exists before creating one) so a re-run is safe.
  Where true idempotency isn't achievable, document clearly (in the flow's
  description or in session notes) which steps are NOT safe to blindly
  re-run, so a person resubmitting a failed run knows what to check first.
- **Scope:** Any flow that creates records, sends communications, or has
  external side effects.
- **Severity trigger:** Any flow likely to be manually resubmitted after a
  failure, or triggered by an event that could plausibly fire more than
  once for the same logical input.
- **Live-search trigger:** Low.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Flow uses a Try/Catch/Finally Scope structure for any meaningful
      external-system logic, not scattered per-action Run-After branches
- [ ] Catch scope's Run After covers failed, skipped, AND timed-out states
      on the Try scope — not just "has failed"
- [ ] Catch/notification logic includes the actual error message, failed
      action name, and a run link — not a generic "something failed" message
- [ ] Retry policy deliberately chosen per action based on idempotency —
      not left at default without consideration for create/send/charge
      actions
- [ ] Exponential backoff favored over aggressive fixed retries for
      throttling (429) scenarios specifically
- [ ] Terminate action with Failed status used when business logic
      determines failure, even if no action technically errored
- [ ] Re-run/idempotency risk considered for any flow with create/send/
      external side effects
- [ ] Per-item Try/Catch used inside Apply to Each loops where individual
      item failures shouldn't block the rest of the batch
