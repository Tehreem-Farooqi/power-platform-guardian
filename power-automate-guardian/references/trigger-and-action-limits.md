# Power Automate Trigger & Action Limits Reference

This is the Power Automate parallel to the Power Apps skill's
`delegation.md` — the category of problem where a flow looks correct but
silently processes only part of the data, or worse, triggers itself into a
runaway loop. Both failure modes are common in AI-generated flows because
neither shows up as a build-time error.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written without this knowledge |
| Why it breaks | The actual mechanism |
| Correct pattern | The fix |
| Scope | Where this applies |
| Severity trigger | When this bites |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. Default row/item limits — different per connector, easy to miss
- **Default habit:** Adding a "Get items"/"Get rows"/"Get files" action and
  assuming it returns everything in the list/table/library.
- **Why it breaks:** Every connector has its own default page size, and
  the action silently returns only the first page unless configured
  otherwise:

  | Connector/action | Default return | Raise via | Hard ceiling |
  |---|---|---|---|
  | SharePoint "Get items" / "Get files" | 100 | `Top Count` up to 5,000, or Pagination beyond that | 5,000 list view threshold per single non-paginated call |
  | Excel Online (Business) "List rows present in a table" | 256 | Pagination | (turn on pagination for full table) |
  | Dataverse "List rows" | 5,000 | Pagination, `Threshold` setting | 100,000 configurable max threshold |
  | Outlook | 25 | **No pagination available on this connector** | 25 |

  A flow built without checking the specific connector's defaults can
  silently process only a fraction of the actual data set, with no error
  — the flow "succeeds," it just didn't do the whole job.
- **Correct pattern:**
  - For counts under 5,000 (SharePoint) or the relevant connector's
    threshold: set `Top Count` (SharePoint) or the equivalent explicit
    count field directly, rather than leaving the default.
  - For counts that may exceed the threshold: turn on **Pagination** in
    the action's Settings, and set an explicit **Threshold** — the number
    entered is the *target* max, not necessarily the exact returned count
    (see #2 for why the actual number can come back higher than expected).
  - **SharePoint specifically:** entering a `Top Count` value greater than
    5,000 causes the flow to **fail outright at runtime**, not silently
    truncate — this is a hard validation limit, not a soft default.
- **Scope:** Universal — check per-connector before assuming a "Get"
  action's behavior generalizes from one connector to another.
- **Severity trigger:** Any flow processing a list/table/library that
  could plausibly exceed the relevant connector's default page size.
- **Live-search trigger:** Low-medium — these specific default numbers
  have been stable for a while, but connector-specific defaults are
  exactly the kind of thing that could be revised; verify if a design
  depends on an exact figure.

## 2. The "threshold rounds up to the page size" confusion
- **Default habit:** Setting a Pagination Threshold to an exact number
  (e.g. 3,000) and expecting exactly that many results back.
- **Why it breaks:** Pagination works by requesting successive **pages**
  from the underlying connector, and each connector has its own internal
  page size (SharePoint: 100 per page: Excel: 256 per page; Dataverse:
  5,000 per page). The Threshold value determines *how many pages* to
  fetch, not an exact cutoff — so the actual returned count gets rounded
  up to the nearest multiple of the connector's page size. This is a
  well-documented source of confusion (real reported case: setting
  Threshold to 3,000 against a file library returned 3,065 items, because
  the underlying page size caused rounding past the requested number).
- **Correct pattern:** If an exact count matters downstream (e.g. a batch
  size assumption, or a loop bound), don't rely on the Threshold number
  being exact — explicitly trim/limit the resulting array in a subsequent
  step (e.g. `take(body('Get_items'), 3000)`) rather than assuming the
  connector returned precisely what was asked for.
- **Scope:** Any flow using Pagination with a Threshold where the exact
  returned count matters for downstream logic.
- **Severity trigger:** Any flow where an off-by-a-few-dozen record count
  would break subsequent logic (e.g. exact batch chunking).
- **Live-search trigger:** Low — this is a structural consequence of
  how pagination works, unlikely to change.

## 3. Filtering at the source instead of after retrieval
- **Default habit:** Retrieving a broad set of items/rows with no filter,
  then using a `Filter array` or `Condition` action afterward to narrow
  it down inside the flow.
- **Why it matters:** This is the Power Automate parallel to Power Apps'
  delegation problem — pulling more data than needed costs more toward
  connector throughput/message-size limits, runs slower, and can push a
  flow toward the default/threshold limits in #1 unnecessarily when a
  source-side filter would have kept the result set small to begin with.
- **Correct pattern:** Use the action's built-in **OData filter query**
  (`$filter` for SharePoint/Dataverse-style connectors) to narrow results
  at the source before they're even returned to the flow, rather than
  retrieving everything and filtering client-side inside the flow.
  Filtering the query is also what makes staying under the 100/5,000-item
  thresholds in #1 realistic for large lists in the first place — a
  filtered query returning 200 relevant rows never has to touch the
  5,000-row ceiling that an unfiltered query against a 50,000-row list
  would.
- **Scope:** Universal, any "Get" action against a filterable connector.
- **Severity trigger:** Any list/table/library large enough that
  retrieving it wholesale would be wasteful or risk hitting a threshold.
- **Live-search trigger:** Low.

## 4. Self-triggering infinite loops — a serious, easy-to-hit trap
- **Default habit:** Using a trigger like SharePoint's **"When an item is
  created or modified"** (or Dataverse's "When a row is added, modified,
  or deleted," or a file-based "When a file is created or modified"
  trigger), and then having the flow itself **update the same
  item/row/file** it was triggered by (e.g. writing back a status,
  timestamp, or calculated value).
- **Why it breaks:** The flow's own update counts as a modification to the
  underlying data source. The trigger — which is watching for exactly that
  kind of change — fires again. The flow runs again, updates the item
  again, and the trigger fires again. **Nothing in the flow designer warns
  about this at build time** — each individual run is perfectly valid on
  its own, so the problem only becomes obvious when someone opens run
  history and finds dozens of runs stacked seconds apart. Left unchecked,
  this burns through API request quotas fast (cross-reference the Power
  Apps skill's `licensing-and-limits.md` — quotas are shared platform-wide),
  can trigger throttling, and can get the flow **automatically disabled**
  by the platform for excessive failure/run rates.
- **Correct pattern — the trigger condition:** Add a **trigger condition**
  in the trigger's settings — a small expression evaluated **before the
  flow starts**, which blocks the run entirely if the condition isn't met.
  This is meaningfully better than checking the same condition *inside*
  the flow (a "flag column" approach), because a trigger condition
  prevents the run from starting at all — it doesn't just make an
  unnecessary run finish faster. A flag-column check still consumes a full
  flow run and still counts against quota; a trigger condition doesn't.
  - **Known limitation:** trigger conditions cannot use the normal dynamic-
    content picker — they must be written as raw expressions referencing
    **internal column names** (cross-reference the Power Apps skill's
    `column-data-type-quirks.md` #1 on SharePoint internal vs. display
    names — the same internal-name requirement applies here).
  - **Common concrete pattern** — block re-trigger once a status has
    already reached a terminal state:
    ```
    @not(or(
        equals(triggerOutputs()?['body/ApprovalStatus'], 'Approved'),
        equals(triggerOutputs()?['body/ApprovalStatus'], 'Rejected')
    ))
    ```
  - **Alternative pattern** — block re-trigger based on who made the last
    change, useful when the flow's own service account/identity is
    distinguishable from real user edits:
    ```
    @not(equals(triggerOutputs()?['body/Editor/Email'], 'flow-service-account@tenant.com'))
    ```
  - A `Filter array` action can also be used to help construct more
    complex trigger conditions, per community-reported technique.
- **This risk is not SharePoint-specific** — it applies equally to
  Dataverse's "When a row is added, modified, or deleted" trigger and to
  file-based "When a file is created or modified" triggers. Check for this
  pattern any time a flow's trigger and its own write-back target could be
  the same record.
- **Scope:** Any flow where the trigger watches for changes to the same
  data source/record the flow itself writes to.
- **Severity trigger:** Any "when created or modified" style trigger
  combined with an Update/Patch action inside the same flow targeting the
  same item — treat this combination as a mandatory check, not an
  edge case.
- **Live-search trigger:** Low — this is a long-standing, well-documented
  platform behavior and mitigation pattern.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Checked the specific connector's default return count and page size
      before assuming a "Get" action returns everything
- [ ] Top Count / Threshold set explicitly where the data set could exceed
      the connector's default (100 for SharePoint, 256 for Excel, 5,000
      for Dataverse) rather than left at default
- [ ] Where an exact downstream count matters, result explicitly trimmed
      rather than trusting Pagination's Threshold to be an exact cutoff
- [ ] Source-side OData filtering used to narrow results before retrieval,
      not just filtered client-side inside the flow after a broad pull
- [ ] Any "when item/row/file created or modified" trigger checked for a
      same-record write-back inside the flow — trigger condition added if so
- [ ] Trigger condition (not just an in-flow flag check) used to prevent
      unnecessary runs and quota consumption, not just unnecessary actions
      within a run that still executes
