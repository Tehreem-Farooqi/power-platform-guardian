# Power Apps Memory & Performance Reference

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written without this knowledge |
| Why it breaks | The actual mechanism — memory, call count, sequential vs parallel |
| Correct pattern | The fix, with real code |
| Scope | Universal / data-source-specific / scale-dependent |
| Severity trigger | The threshold where this stops being cosmetic |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. Batch Patch vs ForAll + Patch
- **Default habit:** `ForAll(colChanges, Patch(Datasource, ThisRecord, {...}))`
  — updates records one-by-one, sequentially.
- **Why it breaks:** Each Patch call waits for the previous to finish.
  Documented real-world cost: ~5 minutes for 500 sequential updates.
- **Fix:** `Patch(Datasource, CollectionOfChanges)` — pass the whole collection
  as the second argument. Power Apps applies all updates simultaneously.
  Collection needs an ID column matching the datasource's key, plus the
  changed columns, with column names matching the datasource exactly.
  ```
  Patch(
      Employees,
      ShowColumns(colUpdateEmployees, "ID", "FullName", "Active")
  );
  ```
- **Scope:** Universal — applies to any tabular data source.
- **Severity trigger:** Matters once you're updating more than a handful of
  records at once; scales badly, fast.
- **Live-search trigger:** Low — this pattern and its performance gap are
  well-established.
- **Skill priority:** HIGH — this is probably the single most common AI-written
  anti-pattern, because ForAll+Patch is the more "obvious" imperative approach.

## 2. Concurrent data loading
- **Default habit:** Sequential `Set()`/`ClearCollect()` calls to independent
  cloud data sources, one after another.
- **Why it breaks:** Each connector call blocks until complete before the
  next starts, even when the calls don't depend on each other.
- **Fix:** Wrap independent cloud calls in `Concurrent()`:
  ```
  Concurrent(
      Set(gblUserProfile, Office365Users.GetUserProfileV2(User().Email)),
      ClearCollect(colActiveProjects, Filter(Projects, ProjectStatus.Value="Active"))
  )
  ```
- **Scope:** Only helps for cloud data source calls — no benefit for
  variables/collections already in memory.
- **Severity trigger:** Matters whenever OnStart/OnVisible loads more than
  one independent data source.
- **Live-search trigger:** Low.

## 3. Delegation as a performance issue (cross-reference)
- See `delegation.md` for the full connector-level breakdown. Delegation
  failures aren't just correctness bugs — non-delegated queries pull large
  amounts of data over the network to the device, which is itself a major
  performance cost even before data-integrity concerns.
- **Escape hatch reminder:** Dataverse views bypass delegation limits — use
  when a needed filter genuinely can't be expressed in delegable Power Fx.

## 4. Caching in collections/variables
- **Default habit:** Re-querying the same cloud data source repeatedly
  across screens/controls instead of loading once and reusing.
- **Why it breaks:** Every cloud read requires a connector round-trip; data
  already in memory is near-instant.
- **Fix:** `ClearCollect(colX, DataSource)` once, reference `colX` thereafter.
- **Caveat:** Collections should be treated as read-only copies, not sources
  of truth. If the data is huge, collection itself becomes the memory problem
  (see #5).
- **Scope:** Universal.
- **Live-search trigger:** Low.

## 5. Oversized collections
- **Default habit:** Collecting a full table with all columns, regardless
  of what's actually used.
- **Why it breaks:** Mobile devices have tight memory ceilings. An oversized
  collection can get the OS to kill the Power Apps process outright — this
  is a crash, not just slowness.
- **Fix:** Use `ShowColumns()` to select only needed columns before
  collecting. For Dataverse, enable **explicit column selection** in
  connection settings so unused table columns aren't even fetched.
  ```
  ClearCollect(colAccounts, ShowColumns(Accounts, "name", "city", "state", "zipcode"))
  ```
- **Design-smell threshold:** Bringing 100k+ records into a collection isn't
  a performance problem to optimize — it's a sign a collection is the wrong
  approach entirely. At that scale, read/write directly against the data
  source, or bring in only the currently-visible subset.
- **Scope:** Universal, more acute for SharePoint/Excel (lower per-row
  overhead tolerance) and on mobile devices specifically.
- **Live-search trigger:** Low.

## 6. OnStart bloat
- **Default habit:** Heavy variable initialization and data loading logic
  in the App object's `OnStart` property.
- **Why it breaks:** Directly delays time-to-first-screen — visible in the
  app's own Analytics > Performance page.
- **Fix:** Move variable initialization to `OnVisible` of the first screen;
  defer further to the specific screen where a variable is actually needed
  if possible. Use `StartScreen` property instead of a `Navigate()` call in
  OnStart. Show a loading spinner/progress bar if OnVisible logic takes more
  than a couple seconds.
- **Scope:** Universal.
- **Live-search trigger:** Low.

## 7. Control count & screen complexity
- **Default habit:** Placing many individual controls on one screen instead
  of using a gallery for repeated elements.
- **Why it breaks:** Every control adds to memory usage at screen load.
  Nested galleries cost far more than a simple text input — Power Apps uses
  an internal, **undocumented** "screen complexity score" rather than a
  published hard control-count limit.
- **Fix:** Use galleries for repeated content — a control inside a gallery
  costs memory once per unique control, not once per rendered row copy.
  Split overly complex screens into multiple screens.
- **Caveat for the skill:** Don't cite a specific control-count number as an
  official limit — Microsoft doesn't publish one, and practitioner reports
  vary (some apps run fine with 400+ controls when isolated/optimized well).
  Frame guidance as "minimize and prefer galleries," not a hard ceiling.
- **Advanced technique:** An **HTML text** control can render complex
  read-only layouts as a single component regardless of internal complexity
  — useful for dense list/gallery views where many individual controls would
  otherwise be needed.
- **Scope:** Universal, more acute on mobile.
- **Live-search trigger:** Low-medium (the complexity-score mechanism itself
  isn't publicly documented in detail).

## 8. DelayOutput on search inputs
- **Default habit:** Text input feeding a gallery filter with default
  `DelayOutput = false`.
- **Why it breaks:** The Text property updates after every keystroke, firing
  a new query (and a new connector call, if not delegated/cached) each time.
- **Fix:** Set `DelayOutput = true` on the text input so the app waits until
  the user stops typing before querying.
- **Scope:** Universal, most important for search-bar-to-gallery patterns.
- **Live-search trigger:** Low.

## 9. Cross-screen control references
- **Default habit:** Referencing a control on a different screen from a
  formula.
- **Why it breaks:** Forces Power Apps to keep that other screen resident in
  memory even when it isn't displayed.
- **Fix:** Store the needed value in a global variable and reference the
  variable, not the control directly.
- **Scope:** Universal.
- **Live-search trigger:** Low.

## 10. The N+1 problem
- **Default habit:** A gallery bound to a table, with each row independently
  doing a `LookUp()` against a related table for a joined field (e.g.
  account name per contact row).
- **Why it breaks:** 1 call to load the gallery + N calls (one per row) =
  N+1 total connector calls. At 100 rows, that's 101 calls for what should
  be one query.
- **Fix — Dataverse:** Relational traversal in a single call —
  `ThisItem.Account.'Account Name'` — Dataverse resolves the relationship
  server-side automatically.
- **Fix — SharePoint (not relational, can't eliminate N+1 the same way):**
  Pre-join locally. Collect both tables before entering the gallery screen,
  then `AddColumns()` to join them into one gallery-ready collection:
  ```
  ClearCollect(colAccounts, Accounts);
  ClearCollect(colContacts, Contacts);
  ClearCollect(
      colGalleryData,
      AddColumns(
          colContacts,
          "AccountName",
          LookUp(colAccounts, ID = AccountID, 'Account Name')
      )
  );
  ```
  Note: the LookUp must reference the **collection** (`colAccounts`), not the
  live data source, or N+1 creeps back in.
- **Scope:** Data-source-specific fix, universal problem.
- **Severity trigger:** Scales linearly with gallery row count — painful
  past a few dozen rows, severe past a few hundred.
- **Live-search trigger:** Low.

---

## Additional real-world flags (practitioner-sourced, not official docs)

- **Connection count:** Microsoft guidance suggests not exceeding ~30
  connections per app (each triggers a separate sign-in prompt, slowing
  startup). Community debate exists over whether this means 30 distinct
  *services* or 30 *connector instances* of the same service (e.g. 30
  SharePoint lists via one login is a different cost profile than 30
  different services). Flag high connection counts as a concern; explain
  the nuance rather than asserting a flat rule.
- **Studio-side lock issue:** Very high SharePoint connection counts in a
  single app (100+) have been reported to cause "ongoing refresh operation"
  locks when editing in Studio — a maker-experience problem distinct from
  runtime performance. Fix: offload logic to Power Automate, reduce
  OnStart code, use Named Formulas instead of variables where possible.
- **Platform-specific load time variance:** Anecdotal reports of significant
  Android vs iOS load-time differences on the same Dataverse-backed app with
  clean OnStart — cause not conclusively identified in available sources.
  Flag as a live-search item if this specific symptom comes up.
