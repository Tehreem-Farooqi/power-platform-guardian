# Power Apps Offline & Mobile Sync Reference

## The decision that shapes everything else
There are **two fundamentally different, non-interchangeable approaches**,
and picking the wrong one for the data source in use is the first and
biggest mistake to catch:

| | Dataverse built-in offline-first | SaveData/LoadData (manual) |
|---|---|---|
| **Data source** | Dataverse tables only | Any connector (commonly SharePoint) |
| **Who owns sync logic** | The platform | The maker — entirely |
| **Conflict handling** | Automatic (column-level, see below) | Entirely hand-built |
| **Code complexity** | No code — toggle + offline profile | Scales up with data complexity |
| **Studio error checking** | Built-in App Checker rules flag config problems | None |
| **Supported Power Fx** | Partial (see limitations below) | All |

**Rule for the skill:** if the app is Dataverse-based, default to
recommending built-in offline-first rather than bolting on a second
SaveData sync system "just in case" — Microsoft explicitly directs
Dataverse-based canvas apps to the built-in feature. Reserve
SaveData/LoadData guidance for apps on SharePoint or other connectors,
where it's genuinely the only offline option available.

## Hard platform constraint — check this first, always
**Offline canvas apps only run through the native Power Apps Mobile
players (iOS, Android, Windows).** A canvas app running in a web browser —
including a mobile device's browser — **cannot run offline, period**. If a
request describes users needing offline access "on their phone" without
specifying the native mobile app, clarify this distinction immediately;
it changes the deployment story entirely (users need the Power Apps Mobile
app installed, not just a browser bookmark).

---

## Route A — Dataverse built-in offline-first

### How it works
Once offline capability is turned on and an offline profile is set (either
**auto-generated** or **custom**), the app always runs offline-first, with
or without a connection. Initial launch requires network access to
download data; after that, all reads/writes happen against local device
storage, and the platform syncs changes to Dataverse automatically in the
background whenever a connection is detected. The maker writes **normal
Power Fx formulas** — no LoadData/SaveData/Connection.Connected branching
needed at all for this route.

### Offline profiles — the actual performance/scope lever
An offline profile determines which **tables, columns, and related
records** get downloaded to the device. This is the primary tool for
controlling both storage footprint and battery/network cost:
- Auto-generated profiles are a fast starting point but pull broadly.
- Custom profiles let you scope aggressively — e.g., only records assigned
  to the current user, only active-status records, only recent records.
- Practitioner guidance for field-deployment scenarios (healthcare,
  manufacturing, field service): filter offline data hard — only cache
  what's assigned to the user, within a recent window (commonly cited
  30–90 days), with active status — and sync at least once per day to
  minimize data loss risk and reduce conflict likelihood.

### Unsupported Power Fx functions/features when offline is enabled
This is a real constraint that changes what you can build, not a minor
footnote:
- `Min`, `Max`, `Average` are **not supported**
- `Relate`, `Unrelate` are **not supported**
- `In` (membership operator) is **not supported**
- `UpdateIf`, `RemoveIf` are **not supported**
- Filtering on a column lookup supports only **one level** of lookup when
  offline is enabled (no multi-hop lookup chains)
- **Many-to-many relationships are not supported** at all
- Microsoft has stated intent to support more of these in future — treat
  this list as current-state, not permanent, and verify live if a specific
  function's offline support status is decision-critical.
- **Skill behavior:** before finalizing any formula for a Dataverse app
  with offline enabled, check it against this list. A formula using
  `Average()` or `UpdateIf()` that works perfectly online will error or
  behave unexpectedly the moment offline mode is active.

### Conflict resolution — how it actually works (not per-record, per-column)
This is a frequently misunderstood mechanism, worth stating precisely:
- Conflict resolution operates at the **column level**, not the record
  level. When a device reconnects, the **last update to each individual
  column** is what gets stored in Dataverse — so if two users offline-edit
  different columns of the same record, both edits can survive; if they
  edit the *same* column, the later sync wins for that column only.
- This means sync **doesn't fail** due to conflicting changes in the way a
  naive "whole record" conflict model might — but it also means a user
  can't assume their entire edit "wins" just because their column choices
  happened to be different from another user's.
- **Server-side plug-ins and validation can still invalidate a synced
  change** even after this column-level resolution — those changes are
  reverted locally, and an error is written to the **Sync Errors** table
  (a standard Dataverse table).
- **Important scope note:** conflict resolution *settings* (configurable
  options) exist in Dataverse but **do not apply to canvas apps** — only to
  model-driven apps. Don't assume a canvas app can be configured with
  different conflict resolution behavior; the column-level last-write-wins
  behavior described above is what canvas apps get.

### Handling sync errors proactively
- Build a model-driven app (or add a view) against the **Sync Error**
  table so errors are visible per-user — Microsoft's own recommendation.
  Selecting a sync error row allows a "Retry changes" action.
- Sync errors show natively on the **Device status page** in model-driven
  apps; canvas apps need this set up manually via the offline template and
  offline status icon.
- For proactive alerting: a cloud flow (Power Automate, Dataverse trigger
  on row added/modified in Sync Errors) can automatically email the
  affected user or push a device notification. Note: retrieving the user's
  email in the flow requires a separate "Get a row by ID" action against
  the Owner column of the Sync Error row — it's not directly available on
  the trigger payload.

---

## Route B — SaveData/LoadData (SharePoint & other non-Dataverse connectors)

### Core functions
```
SaveData(MyCollection, "MyLocalCache")        // Write collection to device storage
LoadData(MyCollection, "MyLocalCache", true)  // Read it back; true = suppress errors if cache doesn't exist yet
```
The third `LoadData` argument is essential for first-run safety — without
it, a fresh install with no existing cache throws an error instead of
gracefully starting empty.

### Storage limits — check before assuming capacity
- **1 MB limit** specifically for apps **embedded in Teams** using
  SaveData/LoadData — images/media are a poor fit for this ceiling.
- Older Microsoft guidance describes this feature as best suited to
  "relatively small quantities of data (dozens of text records)" generally
  not exceeding roughly 2 MB — treat this as a caution about scale, not a
  hard universal cap (mobile native storage is considerably larger than
  this, but the *feature* wasn't designed/optimized for large datasets).
- **No encryption.** SaveData/LoadData do not encrypt stored data — do not
  cache secrets, credentials, or any data the device user shouldn't be able
  to retain locally, even temporarily. This is a real data-handling
  decision point, not a footnote.

### The sync/change-queue pattern (the part makers must build themselves)
Because there's no platform-level sync here, the maker needs an explicit
**dirty/pending-changes pattern** — a way to track which local records have
unsynced edits so they can be pushed later:

```
// App.OnStart — load appropriately based on connectivity
If(
    Connection.Connected,
    // Online: get fresh data, cache it for future offline use
    ClearCollect(colWorkOrders, Filter(WorkOrdersSource, AssignedTo = User().Email));
    SaveData(colWorkOrders, "WorkOrderCache"),
    // Offline: load from local cache
    LoadData(colWorkOrders, "WorkOrderCache", true)
);
LoadData(colPendingChanges, "PendingChanges", true)
```

```
// When a user edits a record while offline — update local collection AND
// queue the change for later sync, rather than only updating locally
UpdateIf(colWorkOrders, ID = ThisItem.ID, {Status: "Complete"});
Collect(colPendingChanges, {ID: ThisItem.ID, Status: "Complete", ModifiedAt: Now()});
SaveData(colWorkOrders, "WorkOrderCache");
SaveData(colPendingChanges, "PendingChanges")
```

```
// Sync when reconnected — wrap in Connection.Connected check and
// error-handle per record (cross-reference error-handling.md)
If(
    Connection.Connected,
    ForAll(
        colPendingChanges,
        IfError(
            Patch(WorkOrdersSource, LookUp(WorkOrdersSource, ID = ID), {Status: Status}),
            // On failure: leave this record in the pending queue for retry, log it
            Trace("Sync failed for record " & ID & ": " & FirstError.Message, TraceSeverity.Error),
            // On success: remove from pending queue
            Remove(colPendingChanges, ThisRecord)
        )
    );
    SaveData(colPendingChanges, "PendingChanges")
)
```
Note: this uses `ForAll`+`Patch` deliberately here rather than Batch Patch
(see `performance.md` #1) — a sync reconciliation loop needs **per-record
error isolation** (one failed sync shouldn't block the others, and each
needs individual retry-queue handling), which a single batch Patch call
doesn't give you. This is a case where the "usually correct" performance
pattern is the wrong tool — flag this nuance rather than reflexively
applying the Batch Patch rule everywhere.

### Conflict handling — entirely the maker's responsibility
Unlike Dataverse's automatic column-level resolution, SharePoint/manual
sync has **no built-in conflict detection at all**. If two users edit the
same record while both are offline, whoever's device syncs last simply
overwrites the other's change with no warning, unless the maker builds
detection themselves. Options to discuss with a client/team, in increasing
order of complexity:
1. **Accept last-write-wins silently** — acceptable only where concurrent
   edits to the same record are genuinely unlikely (e.g. single-user-owned
   records like personal inspection forms).
   2. **Timestamp comparison** — before syncing, compare a `ModifiedAt`
      field from the server record against the timestamp the local copy was
      loaded; if the server's is newer, flag for manual review instead of
   overwriting silently.
3. **Flag-for-review queue** — on detected conflict, don't auto-resolve;
   surface both versions to a user/admin to choose.
**Skill behavior:** always ask which of these is appropriate rather than
defaulting silently to option 1 — this is a real business decision, not
just a technical one, and the "right" answer depends entirely on whether
concurrent edits to the same record are realistically possible in the
app's actual usage pattern.

### Testing pitfalls (these catch nearly everyone at least once)
- **`Connection.Connected` always evaluates to `true` in a browser/Studio**
  — there's no real way to simulate being offline this way. Use a
  placeholder global variable (e.g. `gblSimulateOffline`) during
  development that you control manually, and swap in the real
  `Connection.Connected` reference for the deployed build — or better,
  wrap it: `varConnected: If(gblTestMode, gblSimulateOffline, Connection.Connected)`.
- **SaveData/LoadData throw errors in Studio/browser testing** even when
  the logic is correct — these functions are designed for the native
  device runtime. Expect and ignore these specific errors during
  in-Studio testing; they do not indicate broken code. (An experimental/
  preview capability to enable SaveData/LoadData on the web player has
  existed — verify current status live if browser-testing this is a
  priority, since preview features graduate or get replaced over time.)
- Test actual offline→online transitions on a real device, not just
  Studio preview — the sync/reconnection behavior can't be fully validated
  any other way.

---

## Cross-cutting guidance for both routes
- **Offline is not a checkbox retrofit.** Both routes require deliberate
  design from the start — deciding what data is needed offline, how long
  it can be stale, what happens on conflict, and how errors surface to the
  user. Flag it early in a project's planning if offline support is a
  stated requirement, not as an afterthought once the app is mostly built.
- **Aggressive data scoping matters for both routes** — whether via a
  Dataverse offline profile or manual Filter-before-cache logic, only bring
  down what's actually needed offline. This ties directly to
  `performance.md`'s guidance on oversized collections, doubly so here
  since device storage and sync time are both at stake.
- **Live-search trigger:** Medium-high for this entire file. Offline
  functionality has been explicitly described by Microsoft as "still under
  development" in some capacities, offline profile behavior for Dataverse
  has evolved over recent release waves, and unsupported-function lists
  are stated as likely to shrink over time. Verify current capability
  status live before treating any specific limitation here as permanent.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Confirmed target deployment is the native mobile app, not a browser,
      before promising offline capability
- [ ] Correct route chosen for the data source (Dataverse → built-in
      offline-first; other connectors → SaveData/LoadData) — not mixed
      without a specific reason
- [ ] For Dataverse offline: checked formulas against the unsupported
      function list (Min/Max/Avg, Relate/Unrelate, In, UpdateIf/RemoveIf,
      multi-level lookup filters, many-to-many)
- [ ] For Dataverse offline: offline profile scoped deliberately, not left
      fully auto-generated without review, for anything beyond a trivial app
- [ ] For manual SaveData/LoadData: change-queue/dirty-record pattern
      designed explicitly, not assumed automatic
- [ ] For manual SaveData/LoadData: conflict-handling approach discussed
      and chosen deliberately (silent last-write-wins vs. timestamp check
      vs. flag-for-review), not defaulted silently
- [ ] No secrets/sensitive data cached via SaveData (unencrypted)
- [ ] Testing plan accounts for Connection.Connected always being true in
      Studio/browser, and expects (ignores) SaveData/LoadData errors there
- [ ] Sync error visibility plan in place (Sync Error table view / Device
      status page for Dataverse; explicit retry-queue logging for manual)
