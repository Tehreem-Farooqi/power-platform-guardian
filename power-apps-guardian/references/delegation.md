# Power Apps Delegation Reference

## Core rule
A query is delegable when a Power Fx formula fully translates into an equivalent
query the data source runs itself. Below ~500 records total, non-delegation
doesn't matter — everything processes locally without issue.

Above that: by default Power Apps retrieves only the **first 500 records**
from a non-delegable query (raisable to a max of **2,000** in
File > Settings > Advanced settings > "Data row limit for non-delegable
queries"). A non-delegable formula against a larger data set **silently
returns incomplete results** — no error, no crash, no runtime warning to the
end user. This is a data-integrity bug, not a code-quality nitpick.

The yellow delegation warning triangle in Studio appears for ANY non-delegable
function regardless of actual data size — even on a tiny table. Don't treat
"there's a warning" as automatically severe; check the real/expected record
count of that specific data source before deciding how much it matters.

**Compound formulas need their own check.** An aggregate wrapped around a
filter (e.g. `Sum(Filter(...))`), or a nested `If` wrapping a `Filter`/`Search`,
can silently break delegation even when each individual function looks fine
on its own. Check the whole nested expression, not just each function name.

---

## Reference table schema (applied per connector below)
| Field | Purpose |
|---|---|
| Delegation tier | Full / Partial / None — quick triage |
| Delegable functions & operators | What actually pushes server-side |
| Known non-delegable traps | Function + column-type combos that silently fail |
| Compound formula warnings | Aggregate-of-filter, nested-if patterns, etc. |
| Column-type exceptions | e.g. Person field subfield rules |
| Fix pattern / workaround | The corrected code |
| Production suitability | Is this connector OK for production scale? |
| Live-search trigger | Should the skill re-verify this online rather than trust this doc? |

---

## Microsoft Dataverse
- **Tier:** Full — best-in-class. Virtually all operations push server-side.
- **Delegable:** Filter, Sort, Search, LookUp, First, most comparison operators,
  most aggregates (Sum/Average/Min/Max/CountRows/Count) depending on column type.
- **Traps:** Arithmetic inside the predicate (`Filter(T, field + 10 > 100)`)
  doesn't delegate. `Now()`/`Today()` used inside a comparison don't delegate
  (plain DateTime comparisons otherwise do). `Language()`/`TimeZone()` never delegate.
- **Escape hatch:** Dataverse **views** bypass Power Fx delegation limits
  entirely — `Filter('Table', 'Table (Views)'.'View Name')` runs the view's
  server-defined filter logic directly. Use this when a needed filter genuinely
  can't be expressed as a delegable Power Fx formula.
- **Production suitability:** Recommended default for anything that will scale.
- **Live-search trigger:** Low — this is stable, well-documented ground.

## SQL Server / Azure SQL
- **Tier:** Full — strong, standard SQL operations delegate (Filter, Sort,
  LookUp, most aggregates).
- **Notes:** Platform updates have specifically improved SQL Server/Azure SQL
  delegation limits, and added **virtual tables** (connect external DBs
  without copying data) and **environment variables** for secure connection
  strings — worth checking these are actually being used where relevant,
  since older guidance/training data may under-sell these newer capabilities.
- **Production suitability:** Excellent for large, complex relational data.
- **Live-search trigger:** Medium — delegation limits here have been actively
  improved, worth a periodic freshness check.

## SharePoint
- **Tier:** Partial — the connector most devs get burned by, because
  delegation here depends on THREE things together (function + field type +
  operator) and ALL three must be delegable or the whole query falls back
  to local processing.
- **Traps:**
  - `IsBlank()` does **not** delegate. Fix: `Filter(List, Title = Blank())`
    instead of `Filter(List, IsBlank(Title))`.
  - `Search()` does **not** delegate to SharePoint — only `StartsWith` does
    (not `In`, not `Search`). Text-search UX needs rebuilding around
    `StartsWith`, or offloading true search to a Power Automate flow that
    returns results to the app.
  - Doesn't support `Trim`/`TrimEnds`/`Len`. Does support `Left`, `Mid`,
    `Right`, `Upper`, `Lower`, `Replace`, `Substitute`.
  - Expressions joined by `And`/`Or` delegate; joined by `Not` don't.
  - `GroupBy` and `With` are "hidden" limitations — no warning triangle, but
    they still cap at the record limit.
  - The **ID field** is numeric but only supports `=` for delegation —
    `Filter(List, IsBlank(CustomerId))` will never delegate.
- **Column-type exceptions:** Complex/lookup columns delegate by deferring to
  the subfield. For a **Person** field, only `.Email` and `.DisplayName`
  delegate — nothing else on that field pushes server-side.
- **Record cap nuance:** Non-delegable SharePoint queries are commonly cited
  as bundling/capping at **2,000** records (tied to SharePoint's own list
  view threshold) even where the general app-level setting also maxes at
  2,000 — sources aren't perfectly consistent on this, flagged for live
  verification rather than treated as settled.
- **Getting past 2,000 in a collection:** plain `ClearCollect` only partially
  delegates. Requires deliberate paging (skip/take by an indexed column range)
  to build a full collection beyond the cap.
- **Production suitability:** Fine for simple lists at moderate scale;
  dangerous for anything approaching or exceeding the row cap without
  deliberate workaround patterns.
- **Live-search trigger:** Medium-high — this connector has the most
  practitioner-documented edge cases and periodic updates (delegation support
  itself was added incrementally over time).

## Salesforce
- **Tier:** Full for supported functions, but **predicate support is
  inconsistent** even where the function is technically delegable.
- **Traps:** No `IsBlank` predicate support — `Filter(SalesforceCustomers,
  Name = "Contoso")` delegates, but `Filter(SalesforceCustomers,
  IsBlank(Name))` does not. Same underlying issue as SharePoint's IsBlank
  trap, different mechanism.
- **Production suitability:** Fine, with predicate awareness.
- **Live-search trigger:** Medium — thinner documentation than the "big 3."

## Dynamics 365 (as a data source)
- **Tier:** Full — officially in the delegable connector list, Dataverse-grade
  delegation when accessed as a D365 entity data source.
- **Live-search trigger:** Low.

## Common Data Service (legacy) — ⚠️ DEPRECATED
- **Status:** Deprecated. **Not available in Power Apps** at all — only in
  Logic Apps Standard / Power Automate Premium. Supported by Microsoft only
  until the modern Dataverse connector reaches parity in Logic Apps.
- **Skill behavior:** If a request, an old tutorial, or AI-generated guidance
  references "Common Data Service (Current Environment)," flag it as legacy
  and redirect to the modern **Dataverse** connector. Don't build against it.
- **Live-search trigger:** Low — the deprecation status itself is stable;
  worth re-checking only if working against a very old existing app.

## Excel / OneDrive
- **Tier:** None. No delegation at all — hard-capped at the non-delegable
  row limit (500 default / 2,000 max), full stop.
- **Operational risks beyond delegation:**
  - Excel files can get **locked** while being read/written, which can
    interrupt any Power Automate flows tied to them.
  - A file living in someone's personal OneDrive **disappears if that person
    leaves the company** — a data-source continuity risk, not just a
    performance one.
- **Production suitability:** **Testing/tutorial use only.** If a request
  asks to build a production app against Excel/OneDrive, proactively warn
  rather than silently writing the formula.
- **Live-search trigger:** Low — this is settled, consistent guidance.

## Azure Table Storage
- **Tier:** Treat as **non-delegable by default**. Notably absent from every
  official "delegable data sources" enumeration found (which consistently
  lists only Dataverse/CDS, Dynamics 365, Salesforce, SharePoint, SQL Server).
- **Live-search trigger:** **High** — thin, less-documented ground. Always
  verify live rather than relying on this doc for this specific connector.

---

## Common combo pattern
Many production apps mix SharePoint (simple lookups/lists) with Dataverse
(core transactional data) in the same app. Delegation is evaluated
**per-formula, per-source** — always identify which specific data source a
given formula is touching rather than assuming one connector governs the
whole app.
