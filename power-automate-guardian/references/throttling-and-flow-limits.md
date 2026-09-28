# Power Automate Throttling, Flow Limits & Nested Loop Reference

This file rounds out the concurrency/performance picture started in
`concurrency-and-parallelism.md` with the platform-wide numeric ceilings
that don't depend on any specific setting — the limits that exist whether
or not a flow uses Apply to Each at all, and the two genuinely distinct
layers of throttling that get conflated when a flow starts failing at
volume after working fine in testing.

## The two distinct throttling layers — don't conflate them
- **Platform-level throttling:** Power Automate itself limits how many
  **actions** a flow (or its owner's license) can execute per day,
  regardless of which connector those actions call. This is governed by
  license tier (see #1).
- **Connector-level throttling:** the external service a specific
  connector talks to (SharePoint, Dataverse, a third-party API) imposes
  its own separate rate limit, independent of Power Automate's own
  action-count limits. A flow can be well within its platform-level daily
  action budget and still get throttled by a specific connector's
  per-minute limit (see #2).
- **Why the distinction matters:** the fix for each is different. Platform
  throttling is addressed by license tier/capacity (stacking Process
  licenses, moving to Premium) or by simply doing fewer actions.
  Connector throttling is addressed by respecting that specific
  connector's rate limit (spacing out calls, batching, using retry with
  backoff) — throwing a higher Power Automate license at a SharePoint-
  connector-level 429 won't help, since the constraint is coming from
  SharePoint's own service protection, not from Power Automate's action
  quota.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written/recommended without this knowledge |
| Why it breaks | The actual mechanism |
| Correct pattern | The fix |
| Scope | Where this applies |
| Severity trigger | When this bites |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. Platform-level daily action limits by license tier
- **Default habit:** Building a flow without any sense of how many
  "actions" it consumes per run, or assuming action count only matters for
  performance, not for a hard daily ceiling.
- **The three core tiers (per 24-hour **rolling** window, not calendar
  day):**
  - **Standard:** 6,000 actions/day
  - **Premium:** 40,000 actions/day
  - **Process** (per-flow capacity license): 250,000 actions/day, and
    **multiple Process licenses can be stacked on a single flow**, each
    additional license adding another 250,000/day.
  - A flow's applicable tier follows its **owner's** license (or the
    highest of several licenses the owner holds) — cross-reference
    `alm-and-connections.md` #3: if the owner leaves the organization,
    **the flow reverts to the "Low" performance profile**, which carries
    lower limits across the board (including a lower paginated-items
    ceiling, see #6) — this is a direct throttling consequence of an
    ownership/ALM issue, not a separate root cause, and is exactly why
    `alm-and-connections.md`'s ownership continuity guidance matters for
    performance, not just for access continuity.
- **What actually counts as one "action":**
  - Every connector action, every HTTP action, and every built-in action
    (including something as simple as Initialize Variable or Compose)
    counts.
  - **Both successful and failed** actions count.
  - **Retries and pagination requests count as separate actions** — a
    heavily-retried or heavily-paginated flow burns through quota faster
    than its visible action count in the designer would suggest.
  - **An action inside a loop counts once per iteration** — an action
    inside an Apply to Each processing 500 items generates 500 counted
    actions for that single designer-visible step, not one.
  - **Skipped actions do not count** (e.g., the untaken branch of an
    if/else).
- **Correct pattern:** For any flow processing meaningful volume, estimate
  actual daily action consumption (accounting for loop multiplication —
  see #7 for why nested loops make this estimate much larger than it
  looks) against the owner's license tier **before** assuming the flow
  will scale, not after it starts getting throttled in production.
- **Scope:** Universal — every flow consumes against this budget.
- **Severity trigger:** Any flow with loops processing more than a
  handful of items, or any flow expected to run at meaningful frequency/
  volume in production.
- **Live-search trigger:** Medium — these specific numbers (6,000/40,000/
  250,000) are current platform documentation but are exactly the kind of
  licensing-adjacent figure Microsoft has revised before (cross-reference
  the Power Apps skill's `licensing-and-limits.md` volatility note) —
  verify live before treating them as permanent.

## 2. Connector-specific service protection limits
- **Default habit:** Assuming a connector (e.g., SharePoint) has no rate
  limit of its own beyond Power Automate's general action quota.
- **Why it breaks:** Individual connectors impose their own throttling as
  a service-protection mechanism, **separate from and in addition to**
  Power Automate's platform-level action limits. Documented example:
  SharePoint enforces **600 API calls per connection per 60 seconds** —
  and critically, this is **per connection, not per flow**: multiple
  flows sharing the same underlying SharePoint connection share that
  600/minute budget collectively, so a spike in one flow's activity can
  throttle a completely different flow using the same connection.
- **Early warning signal:** once usage passes roughly 80% of a connector's
  per-window limit, some connectors (SharePoint among them) return
  `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset` headers
  on **successful** responses, not just on the eventual 429 — a flow or
  monitoring process that inspects these headers can back off proactively
  before actually getting throttled, rather than reactively handling a
  429 after the fact.
- **Correct pattern:** Check the specific connector's own documentation
  page for its "Throttling" section before assuming a flow scales — most
  connector reference pages document this explicitly. Design around the
  specific connector's limit (batching, spacing calls, or consolidating
  multiple flows' usage of a shared connection) rather than only tuning
  Power Automate-level settings, which won't help against a connector-
  level 429.
- **Scope:** Any connector with documented service-protection limits —
  check per-connector, don't assume the SharePoint figure generalizes.
- **Severity trigger:** Any flow (or set of flows sharing a connection)
  making high-volume calls to a single connector in a short window.
- **Live-search trigger:** Medium-high — connector-specific throttling
  numbers are set by the underlying service (SharePoint, in the example)
  and can change independent of any Power Automate platform update;
  verify the specific connector's current documented limit live.

## 3. Flows getting automatically disabled — retention/health limits
- **Default habit:** Assuming a flow that starts failing or goes quiet
  will simply sit there indefinitely until someone notices and fixes it.
- **Why it matters:** Power Automate **automatically disables** flows
  under several conditions, which can be mistaken for a manual action or
  a separate bug if the person investigating doesn't know these exist:
  - **Continuous errors:** a flow whose trigger or actions fail
    continuously gets turned off after **14 days**.
  - **No trigger activity:** a flow not triggered within a **90-day**
    period may be turned off — **except** flows owned by users with
    premium licenses or assigned Process (per-flow) capacity licenses,
    which are exempt from this specific suspension. Owners/co-owners are
    notified 30 days before suspension and can reactivate it.
  - **Consistent throttling:** a flow that is consistently throttled
    (cross-reference #1/#2) gets turned off after **14 days** — dedicating
    Process license capacity to the flow is the documented mitigation.
- **Correct pattern:** Treat these as real production risks to design
  around, not edge cases: ensure error handling (per
  `error-handling-and-retry.md`) prevents genuinely benign, expected
  failures from counting as the kind of continuous errors that trigger
  auto-disable, and flag infrequently-triggered-but-still-important flows
  as candidates for a premium/Process license specifically to avoid the
  90-day suspension risk if that's a realistic scenario for the flow's
  actual usage pattern.
- **Scope:** Universal — applies to every flow regardless of complexity.
- **Severity trigger:** Any flow expected to run infrequently, any flow
  prone to occasional connector-level throttling, or any flow whose
  error rate isn't tightly controlled.
- **Live-search trigger:** Medium — these specific day-counts are current
  documented policy but are the kind of operational parameter Microsoft
  could adjust; verify live if a specific flow's continuity depends on
  exact timing.

## 4. Structural per-flow limits — 500 actions, 8 levels of nesting
- **Default habit:** Building one large, flat (or deeply nested) flow
  without considering the flow's own structural ceilings.
- **Why it matters:** A single flow definition is limited to **500
  actions**, and **8 levels of allowed nesting depth for actions** —
  both documented, hard structural limits, not soft performance
  guidance. Microsoft's own guidance explicitly recommends **child
  flows** as the mechanism to work around either limit: split a large
  flow into child flows to stay under 500 actions, or to go beyond 8
  levels of nesting.
- **Additional practical note:** flows with a large number of actions can
  encounter **editing performance issues in the designer itself** even
  before hitting the 500-action hard limit — a flow approaching a few
  hundred actions is worth considering for a child-flow split on
  maintainability grounds alone, independent of the hard ceiling.
- **Correct pattern:** For any flow growing large or deeply nested, plan a
  child-flow decomposition proactively — by logical sub-process, not just
  reactively once a limit is hit. This also has an ALM upside worth
  cross-referencing `connector-file-locking.md`'s comment-thread-
  reported pattern of using a child flow specifically for isolated
  retry logic (cleaner run history, and separately licensable/capacity-
  assignable via Process licenses per #1).
- **Scope:** Any flow growing beyond a modest number of actions or
  nesting levels.
- **Severity trigger:** Approaching either the 500-action or 8-level
  nesting ceiling — plan the split well before hitting the hard limit,
  since restructuring an already-large flow under time pressure is worse
  than designing for decomposition from the start.
- **Live-search trigger:** Low — these are stable, long-documented
  structural limits.

## 5. Do Until — non-intuitive default behavior and limits
- **Default habit:** Adding a Do Until loop without configuring its
  Count/Timeout settings, assuming sensible defaults will "just work" for
  whatever the actual wait scenario requires.
- **Why it matters:**
  - **Default limits:** Count defaults to **60**, Timeout defaults to
    **PT1H** (1 hour, in ISO 8601 duration format) — the loop stops at
    whichever limit is hit **first**. For a scenario needing to poll for
    longer than an hour or more than 60 times, the default silently cuts
    the wait short, and the flow proceeds past the loop as if the
    condition were met (or with whatever fallback logic follows) even
    though it wasn't.
  - **Configurable maximums:** Count can be raised to a maximum of
    **5,000**; Timeout can be raised to a maximum of **P30D** (30 days).
  - **The loop always runs at least once, non-intuitively** — Do Until
    evaluates its condition at the **end** of each iteration, not the
    beginning, so the actions inside execute at least one time even if
    the condition was already true before the loop started. This differs
    from how a "while" loop works in most general-purpose languages and
    is a common source of off-by-one confusion.
  - **The 30-day absolute ceiling is a real architectural constraint** —
    for genuinely long-running processes (e.g., an approval that could
    plausibly take longer than 30 days), a polling Do Until loop is
    structurally the wrong tool regardless of configuration, since no
    setting can exceed the 30-day maximum.
- **Correct pattern:**
  - Set Count and Timeout deliberately based on the actual expected wait
    scenario, not left at default, for any Do Until loop.
  - Account for the "runs at least once" behavior explicitly when the
    condition might already be true at loop entry.
  - For genuinely long-running processes that could exceed 30 days,
    don't force-fit a Do Until — use an event/trigger-driven architecture
    instead (e.g., a separate flow triggered by the eventual approval
    event, or a scheduled flow that periodically checks persisted state
    in a data source rather than blocking inside one long-running flow
    instance).
- **Scope:** Any flow using Do Until.
- **Severity trigger:** Any Do Until loop whose actual required wait
  duration or iteration count isn't clearly well under the defaults (60
  iterations / 1 hour) — verify explicitly rather than assume the default
  is adequate.
- **Live-search trigger:** Low — these specific limits are stable,
  long-documented behavior.

## 6. Paginated items ceiling varies by performance profile
- **Default habit:** Assuming the pagination threshold ceiling (see
  `trigger-and-action-limits.md` #1) is a flat number regardless of the
  flow's performance profile.
- **Why it matters:** The maximum paginated items ceiling is **5,000 for
  the "Low" performance profile** and **100,000 for all other**
  performance profiles. Cross-reference #1's note: a flow can drop to the
  Low performance profile if its owner leaves the organization — meaning
  a flow that used to support pagination up to 100,000 items can silently
  drop to a 5,000-item ceiling purely as a consequence of an ownership
  change, with no code or configuration change to the flow itself. This
  is a direct, easy-to-miss link between the ALM concern in
  `alm-and-connections.md` and a functional data-processing ceiling.
- **Correct pattern:** When diagnosing a flow that used to paginate
  successfully past 5,000 items and no longer does, check the flow's
  current performance profile/owner status before assuming a logic bug —
  cross-reference `alm-and-connections.md`'s ownership-continuity
  guidance as the actual fix (reassign to an active owner with an
  adequate license) rather than debugging the pagination logic itself.
- **Scope:** Any flow relying on pagination beyond 5,000 items.
- **Severity trigger:** Any flow whose owner's employment/license status
  is uncertain or has recently changed, combined with reliance on
  high-volume pagination.
- **Live-search trigger:** Low-medium.

## 7. Nested loops — the multiplication effect on action counts and runtime
- **Default habit:** Nesting an Apply to Each inside another Apply to
  Each (or inside a Do Until) without calculating the actual multiplied
  action count and runtime this produces.
- **Why it breaks:** Actions inside nested loops multiply, not add. An
  outer loop of 100 items containing an inner loop of 50 items, with 2
  actions inside the inner loop, produces **100 × 50 × 2 = 10,000**
  counted actions for that nested block alone — easily enough on its own
  to exceed a Standard license's entire 6,000/day budget (cross-reference
  #1) in a single run, well before considering anything else the flow
  does. The same multiplication applies to runtime: nested sequential
  loops compound latency multiplicatively, which is a large part of why
  concurrency tuning (per `concurrency-and-parallelism.md`) matters so
  much more for nested structures than flat ones.
- **Correct pattern:**
  - Before building a nested loop, calculate the worst-case multiplied
    action count against the flow's license tier — don't discover the
    multiplication effect only after hitting a throttling or auto-disable
    condition (#1/#3) in production.
  - Where possible, restructure to avoid true nesting: flatten data
    before looping (e.g., use `AddColumns`/join-style techniques,
    cross-referencing the Power Apps skill's `component-reuse.md`/
    `performance.md` philosophy of pre-joining data rather than nesting
    lookups) so a single loop processes pre-combined data rather than one
    loop containing another.
  - Where nesting is genuinely unavoidable, evaluate whether the inner
    loop's work belongs in a **child flow** instead — this doesn't reduce
    the total action count, but it does keep the parent flow under the
    500-action/8-level structural limits (#4) and can make the action
    consumption easier to reason about and monitor per sub-process.
- **Scope:** Any flow with a loop nested inside another loop.
- **Severity trigger:** Any nested-loop structure — treat calculating the
  multiplied action count as a mandatory step before finalizing the
  design, not an optional check.
- **Live-search trigger:** Low — the multiplication mechanism itself is a
  stable structural consequence of how loops and action-counting work.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Distinguished platform-level throttling (license-tier action
      quota) from connector-level throttling (service-specific, e.g.
      SharePoint's 600/min per connection) before diagnosing a 429
- [ ] Estimated actual daily action consumption — accounting for loop
      multiplication — against the flow owner's license tier
- [ ] Checked the specific connector's documented throttling limits
      rather than assuming Power Automate's platform quota is the only
      constraint
- [ ] Considered auto-disable risk (14-day continuous errors, 90-day no
      trigger activity, 14-day consistent throttling) for the specific
      flow's expected usage pattern
- [ ] Planned child-flow decomposition proactively for flows approaching
      500 actions or 8 levels of nesting, not reactively
- [ ] Do Until Count/Timeout set deliberately for the actual scenario,
      not left at default (60 / 1 hour), with the "runs at least once"
      behavior accounted for
- [ ] Long-running processes (>30 days) architected as event/trigger-
      driven rather than forced into a Do Until polling loop
- [ ] Nested loops' worst-case multiplied action count calculated before
      finalizing the design, with flattening or child-flow decomposition
      considered as alternatives to true nesting
- [ ] Flow owner/license status checked when diagnosing unexplained drops
      in pagination ceiling or throttling behavior (cross-reference
      `alm-and-connections.md`)
