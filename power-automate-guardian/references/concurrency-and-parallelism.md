# Power Automate Concurrency & Parallelism Reference

There are **two distinct, easily-confused concurrency settings** in Power
Automate, and mixing them up is itself a common mistake worth flagging
before anything else in this file:

| | Trigger concurrency control | Apply to Each concurrency (Degree of Parallelism) |
|---|---|---|
| **Controls** | How many separate **flow runs** execute simultaneously | How many **iterations within one loop, in one run** execute simultaneously |
| **Where set** | Trigger's own Settings | The specific Apply to Each action's Settings |
| **Range** | 1–100 | 1–50 |
| **Default** | Sources vary on the exact default (see #4) | Off/1 (fully sequential) |
| **Setting to 1** | Forces the whole flow to run as a single serial instance — a second triggering event waits for the first run to finish entirely | Forces that specific loop to process one item at a time |

Both matter for the same underlying reason — concurrent execution touching
shared state — but they operate at different scopes and need to be
reasoned about separately.

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

## 1. Leaving Apply to Each fully sequential when performance matters
- **Default habit:** Not touching the concurrency setting at all, leaving
  every Apply to Each loop processing one item at a time regardless of
  loop size.
- **Why it matters:** The performance gap is dramatic and well-documented
  — a measured real-world test processing 1,000 records took roughly 4
  minutes sequentially versus about 15 seconds with Degree of Parallelism
  set to 30. For loops with more than a handful of items and independent,
  side-effect-safe iterations, leaving this off is a real, avoidable
  performance cost — directly parallel to the Power Apps skill's
  `performance.md` #1 (Batch Patch vs. ForAll+Patch) in spirit: a
  sequential default that's fine for small counts becomes a serious
  bottleneck at scale.
- **Correct pattern:** Enable Concurrency Control and set an appropriate
  Degree of Parallelism **once the loop's iterations are confirmed
  independent** (see #2 and #3 for when they aren't). Rough starting
  points reported by practitioners (treat as informal guidance, not an
  official Microsoft recommendation):
  - Small datasets (<100 items): 5–10
  - Medium datasets (100–1,000 items): 10–20
  - Large datasets (>1,000 items): 20–50, monitored carefully for
    throttling (see #5)
- **Scope:** Any Apply to Each loop processing more than a small number of
  items, where iteration independence has been confirmed.
- **Severity trigger:** Loops processing dozens or more items where each
  iteration involves a network call (connector action) — the per-iteration
  latency is what compounds into the dramatic sequential-vs-parallel gap.
- **Live-search trigger:** Low — the mechanism and general magnitude of
  the performance difference is stable.

## 2. Enabling parallelism on a loop that manipulates shared variables
- **Default habit:** Turning on Degree of Parallelism for performance
  without checking whether the loop body reads or writes a
  Set/Increment/Append-to-array-variable action referencing a variable
  **outside** the loop's own iteration scope.
- **Why it breaks:** Power Automate's variable actions (Increment
  variable, Compose, Append to array/string variable, etc.) are **not
  thread-safe** across parallel iterations. When multiple iterations read
  and write the same variable at nearly the same time, you get a race
  condition — the final value depends on unpredictable timing, and updates
  can be silently lost (e.g. an Increment variable counter that should
  reach 1,000 after 1,000 iterations landing at some lower number because
  concurrent increments overwrote each other).
- **Hard platform restriction, not just a risk:** once Concurrency Control
  is enabled on an Apply to Each, **you cannot use array variables as
  accumulators inside that loop** — this is a designer-level restriction,
  not something that merely behaves badly; plan the loop's data-collection
  strategy around this before enabling concurrency, not after hitting the
  restriction mid-build.
- **Correct pattern:**
  - Simplest and most reliable: **keep Degree of Parallelism at 1 (off)**
    whenever the loop body manipulates a variable shared across
    iterations, if the loop is small enough or the performance cost is
    acceptable. This is often the right call for infrequent or small-batch
    flows — parallelism isn't free to reason about, and isn't worth the
    complexity for a loop of 20 items running once a day.
  - If parallelism is genuinely needed for performance: eliminate the
    shared mutable state entirely rather than trying to make it safe.
    Options: write each iteration's result to a **distinct** location
    (e.g., a per-item record in a data source, not a shared in-memory
    variable), or restructure to avoid needing a running accumulator
    inside the parallel loop at all — aggregate afterward from the
    individual results instead of accumulating during the loop.
- **Scope:** Any Apply to Each loop containing variable-manipulation
  actions referencing a variable outside the current iteration.
- **Severity trigger:** Any loop with Increment/Append/Compose-to-shared-
  variable patterns — check this before enabling concurrency, not after.
- **Live-search trigger:** Low — this is a structural platform behavior,
  stable and unlikely to change.

## 3. Enabling parallelism on order-dependent iterations
- **Default habit:** Enabling concurrency on a loop where each iteration's
  logic depends on the previous iteration's output or resulting state
  (e.g. a flow that pages through an API where each request needs a token
  or cursor returned by the previous request).
- **Why it breaks:** Parallel execution has no guaranteed ordering — if
  iteration N genuinely depends on iteration N-1 having already completed
  and produced a specific result, running them concurrently breaks the
  dependency outright, not just riskily.
- **Correct pattern:** Recognize order-dependent logic before enabling
  concurrency at all — this isn't a tuning problem, it's a structural
  incompatibility. For workloads that are *mostly* independent but need
  some ordering constraint (e.g. process in batches, but items within a
  batch don't depend on each other), use a **hybrid batching pattern**:
  split the full set into batches (e.g. 2,000 items into 20 batches of
  100), run the **batches** serially (preserving whatever ordering
  requirement exists between batches), and run each batch's **inner**
  items in parallel (since those don't depend on each other).
- **Scope:** Any loop with a genuine sequential dependency between
  iterations — pagination-style API calls being the most common real case.
- **Severity trigger:** Any loop processing paginated results, cursor-based
  API calls, or any logic explicitly described as "do X, then based on
  that result do Y for the next item."
- **Live-search trigger:** Low.

## 4. Shared-resource race conditions beyond variables — duplicate records,
   conflicting writes
- **Default habit:** Enabling parallelism on a loop where each iteration
  independently checks a condition and then acts on it (a common
  "check-then-create" or "check-then-update" pattern), assuming the check
  and the action stay atomically paired per iteration.
- **Why it breaks:** Two classic race conditions worth naming explicitly:
  - **Duplicate record creation:** two parallel iterations both check
    "does a record matching X already exist?" at nearly the same moment,
    both see "no" (because neither has created it yet), and both proceed
    to create it — resulting in a duplicate that a purely sequential loop
    would never have produced, since the check and creation are no longer
    effectively atomic across iterations.
  - **Conflicting writes to the same underlying resource:** if multiple
    parallel iterations write to the *same* file, record, or list item
    (not just the same in-flow variable — an actual external data source
    row), there's no guaranteed order to those writes, and no reliable way
    to know which iteration's write "won." Cross-reference the Power Apps
    skill's `offline-and-mobile-sync.md` conflict-resolution concepts —
    the underlying problem (concurrent writers, no coordination) is the
    same shape, just without Dataverse's column-level automatic resolution
    to fall back on here.
- **Correct pattern — pick based on what's actually needed:**
  - **Partition the work** so parallel branches never touch the same
    record — e.g., partition by a natural key (region, category, ID range)
    so each parallel branch owns a disjoint subset of records.
  - **Database-level atomicity** — where the underlying data source
    supports an atomic "create if not exists" or conditional-update
    operation, use it instead of a separate check-then-act pair of steps.
  - **Optimistic concurrency with conflict detection** — read a version/
    timestamp, attempt the write conditioned on that version being
    unchanged, and handle the conflict explicitly (retry, merge, or flag)
    if the condition fails — conceptually the same idea as the Power Apps
    offline-sync timestamp-comparison pattern.
  - **Mutex-style locking** — for genuinely serialized access to a shared
    resource, an explicit lock record (in Dataverse or Azure Storage, for
    example) that an iteration must acquire before proceeding and release
    afterward. Adds real complexity — reserve for cases where the other
    options genuinely don't fit, not as a default approach.
- **Scope:** Any parallel loop with check-then-act logic against a shared
  external data source.
- **Severity trigger:** Any parallelized loop that creates records
  conditionally, or writes to a resource more than one iteration could
  plausibly touch.
- **Live-search trigger:** Low-medium — the concepts are stable software-
  engineering concurrency patterns; specific Dataverse/Azure Storage
  locking mechanics could evolve, worth verifying if implementing the
  mutex approach specifically.

## 5. Parallelism amplifying throttling risk
- **Default habit:** Cranking Degree of Parallelism to the maximum (50)
  for a large dataset, assuming higher is strictly better for performance.
- **Why it breaks:** Cross-reference `error-handling-and-retry.md` #3 and
  the Power Apps skill's `licensing-and-limits.md` — every action
  execution counts toward a connector's API quota, and parallel execution
  means many more requests fired in a short window. Raising parallelism
  can push a flow past a connector's throttling limits faster than
  sequential execution would, turning a "faster flow" into a "flow that
  gets throttled and needs its retry logic to absorb the fallout" —
  meaning the retry policy design in `error-handling-and-retry.md` isn't
  optional once meaningful parallelism is introduced, it's a prerequisite.
- **Correct pattern:** Increase Degree of Parallelism incrementally and
  monitor for throttling (429) responses rather than jumping straight to
  50 for a large dataset. Ensure exponential-backoff retry policies (per
  `error-handling-and-retry.md` #3) are in place on the actions inside a
  parallelized loop before increasing parallelism, not after hitting
  throttling in production.
- **Scope:** Any parallelized loop calling a connector with meaningful
  per-day/per-minute quota limits.
- **Severity trigger:** Any Degree of Parallelism setting above roughly 10
  against a connector with known throttling behavior — treat as the point
  where retry-policy readiness should be explicitly confirmed.
- **Live-search trigger:** Low-medium — the general dynamic is stable, but
  specific per-connector throttling thresholds shift (cross-reference
  `licensing-and-limits.md`'s note on API quota volatility).

## 6. Trigger-level concurrency — a separate lever, often overlooked
- **Default habit:** Only ever thinking about concurrency in terms of
  Apply to Each, without realizing the trigger itself has its own,
  separate concurrency setting governing how many **whole flow runs**
  execute at once.
- **Why it matters:** Even with a perfectly safe, non-parallel Apply to
  Each inside a flow, **multiple separate triggering events can still
  cause multiple flow runs to execute concurrently**, each potentially
  touching overlapping data — this is a distinct race-condition surface
  from anything happening inside a single run's loop.
- **Documented range and default — note the inconsistency across
  sources:** the trigger concurrency setting's range is commonly cited as
  1–100. Reported defaults vary across sources (some cite 25, and the
  visible slider default in some contexts has been reported differently)
  — **this is exactly the kind of detail to verify live rather than trust
  a cached number for**, since UI defaults are the sort of thing Microsoft
  adjusts between releases.
- **Correct pattern:** For flows where two triggering events touching
  related/overlapping data could race with each other (e.g. two people
  updating related SharePoint items in quick succession, each firing the
  same flow), consider setting trigger concurrency to **1** to force fully
  serial execution across runs — trading throughput for guaranteed
  ordering and avoiding cross-run race conditions that Apply to Each
  settings alone can't prevent, since Apply to Each only governs
  concurrency *within* a single run.
- **Scope:** Any flow whose trigger could plausibly fire multiple times in
  close succession for related data.
- **Severity trigger:** Any flow triggered by an event source with
  realistic potential for near-simultaneous related triggers (e.g. a busy
  SharePoint list with multiple concurrent editors).
- **Live-search trigger:** Medium-high specifically for the exact default
  value and range — verify current UI behavior before citing a specific
  default number.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Distinguished trigger-level concurrency from Apply to Each
      concurrency — not conflated as the same setting
- [ ] Confirmed loop iterations are independent (no shared variable
      manipulation, no order dependency) before enabling Apply to Each
      concurrency
- [ ] If shared variables are needed, restructured to avoid them rather
      than enabling concurrency and hoping for the best — remembered array
      variables as accumulators are outright disallowed once concurrency
      is on
- [ ] Order-dependent loops (pagination, cursor-based APIs) either kept
      sequential or restructured into a serial-batches/parallel-within-
      batch hybrid
- [ ] Check-then-act patterns against shared external data evaluated for
      duplicate-creation or conflicting-write race conditions; partitioning
      or atomic operations used where the risk is real
- [ ] Retry policy with exponential backoff confirmed in place before
      increasing Degree of Parallelism meaningfully, to absorb throttling
      risk rather than let it cause failures
- [ ] Trigger-level concurrency considered explicitly for flows with
      realistic risk of near-simultaneous related trigger events
