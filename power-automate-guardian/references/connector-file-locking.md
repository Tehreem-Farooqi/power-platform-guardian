# Power Automate Connector File-Locking & Session Reference

This file exists because of a real production incident, not a hypothetical:
an Excel Online (Business) "List rows present in a table" action left a
file locked long enough that a subsequent "Move file" step failed
intermittently, with no obvious pattern until the underlying cause (a
lingering co-authoring session) was identified. This is a genuinely
under-documented class of problem — the error message doesn't point at the
real cause, and AI-generated flows have no way to know about it unless
told explicitly.

## The core mechanism (why this happens at all)
Several Office file-format connectors (Excel Online, and — per practitioner
reports — Word Online) don't just read/write a file and release it
immediately. Actions like "List rows present in a table" open something
functionally equivalent to a **live co-authoring session** against the
file, the same as if a person had opened it in the browser. That session
is not guaranteed to close the moment the action's output returns — it can
persist afterward, and while it persists, other operations against the
same file (including a **Move** or **Delete** from a *different*
connector, like SharePoint) can fail with a lock error.

This is why the failure looked "random" in the original report: it wasn't
random at all — it was a race between an undetermined session-teardown
time and the next step's execution, and the next step usually loses that
race unless something is done about it deliberately.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written without this knowledge |
| Why it breaks | The actual mechanism |
| Correct pattern | The fix, with real approach |
| Scope | Which connector(s)/file type(s) this applies to |
| Severity trigger | When this actually bites |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. Excel connector holding a file lock after the action completes
- **Default habit:** Reading a table from an Excel file (`List rows present
  in a table`, or similar Excel actions) and immediately performing a
  file-level operation (Move, Delete, Update file properties) on that same
  file in a later step, with no gap or handling in between.
- **Why it breaks:** The Excel connector's session on the file isn't
  guaranteed to release the instant the action returns.
- **Documented lock durations — note these DIFFER by connector variant:**
  - **Excel Online (Business)** connector (SharePoint/OneDrive for
    Business libraries, Microsoft Graph-based): Microsoft's own connector
    documentation states a file **may be locked for an update or delete up
    to 6 minutes** since the last use of the connector.
  - **Excel Online (OneDrive)** connector (a distinct, separate connector):
    Microsoft's documentation for this variant states up to **12 minutes**.
  - Practitioner reports (including the incident this file is built from)
    cite figures anywhere from a few minutes up to 10–13 minutes in
    practice — consistent with "up to N minutes" being a ceiling, not a
    fixed duration; actual release time varies by file size, action type,
    and load.
  - **Skill behavior:** don't assume a single universal number. Identify
    which specific Excel connector variant is in use (Business vs.
    OneDrive) before citing a duration, and treat even the documented
    figures as ceilings that can occasionally be exceeded in practice
    based on community reports.
- **Confirmed NOT unique to Excel:** practitioner reports confirm the same
  category of issue occurs with **Word documents** in SharePoint libraries
  — a flow updating metadata after some Word-related operation can hit the
  identical "locked for shared use" error. Treat any Office-format
  connector session (Excel, Word, and plausibly PowerPoint) as carrying
  this risk, not just Excel specifically — verify live for a specific
  format if this matters for a given flow.
- **Scope:** Excel Online (Business), Excel Online (OneDrive), and by
  practitioner report, Word-related connector actions against SharePoint/
  OneDrive-hosted files.
- **Severity trigger:** Any flow that performs a file-level operation
  (Move, Delete, Copy that overwrites, Update file properties) on the same
  file within the same run, shortly after an Excel/Word connector action
  touched that file.
- **Live-search trigger:** Medium — the mechanism is stable and long-
  documented, but exact durations have shifted before and differ by
  connector variant; verify current stated limits if a specific number
  matters for SLA/timing design.

## 2. The fix that addresses root cause, not symptom: never let the risky
   connector touch the file you need to move/delete
- **This is the pattern from the original incident, confirmed as a
  documented "Solution #3" pattern by independent practitioner sources
  (Matthew Devaney's "4 Solutions" writeup, corroborating the approach
  independently arrived at in the reported incident):**
  1. Copy the file to a temporary/staging folder first.
  2. Read the table data from the **copy**, never the original.
  3. Delete the temporary copy once reading is complete.
  4. The original file — never touched by the Excel connector at all — is
     immediately free to be moved/deleted/renamed with no lock conflict.
- **Why this is better than waiting out the lock:** the lock's actual
  release time isn't fixed or guaranteed (see #1's duration variance) — a
  fixed delay is a bet on a number that can be wrong under load, while
  never opening the session on the original file at all removes the race
  condition entirely rather than trying to time around it.
- **Retry policy still needed for the copy/delete steps themselves** — a
  separate, much shorter-lived lock can occur if two runs process the same
  file at nearly the same moment (e.g. one triggered by direct upload, one
  by a folder-watch trigger). This lock clears in seconds; a short
  automatic retry (see `error-handling-and-retry.md` once built) fully
  resolves it. This is a distinct, minor risk from the main 6-12 minute
  Excel session lock — don't conflate the two when diagnosing.
- **When this pattern doesn't fully apply — writing, not just reading:**
  for flows that need to **fill in/update** an Excel template rather than
  just read it, the equivalent pattern (per Devaney's "Solution #4") is:
  copy the file to a temp folder, perform the update there, copy the
  *completed* file to the real output folder, then delete the temp copy.
  The original/template file is still never subjected to the lock-then-
  move race.
- **Scope:** Any flow structure where an Excel/Word read or write is
  followed by a file-management operation on the same file.
- **Severity trigger:** Any flow with this read-then-move/delete shape —
  treat this as the default pattern to reach for, not a fallback after
  something else fails.
- **Live-search trigger:** Low — this is a well-corroborated pattern from
  multiple independent sources.

## 3. Alternative/supplementary fixes worth knowing (not always the best
   first choice, but useful in different circumstances)
- **Loop-until-unlocked (Do Until + Configure Run After + Delay):**
  attempt the file operation; if it fails with status 400 (locked), delay
  30 seconds and retry; repeat until success. Documented to work but can
  take many iterations (one practitioner reported 20 loop iterations
  before success) — functional, but slower and noisier (each failed
  attempt still shows as a failure in run history unless Run After is
  configured correctly) than the copy-first approach in #2. Reasonable
  when the flow's structure can't easily avoid touching the original file.
- **Bypass the lock entirely for deletion (SharePoint HTTP trick):** for
  the specific case of **deleting** a locked file, a SharePoint "Send an
  HTTP request" action to the `.../recycle` endpoint with a
  `Prefer: bypass-shared-lock` header ignores the lock and deletes
  immediately. This only solves deletion — it does not help if the goal is
  to move, read, or update the file rather than remove it.
- **Check out file / Discard check out (community-reported, verify live):**
  adding an explicit "Check out file" action before the Excel/Word action
  and "Discard check out" after has been reported by practitioners as
  resolving the issue in some cases. This is a community-reported
  workaround, not officially documented Microsoft guidance for this
  specific purpose — verify current behavior live before relying on it as
  a primary fix, and prefer the copy-first pattern (#2) as the more
  robustly corroborated approach.
- **Child flow retry isolation:** running the risky Excel/Word operation
  inside a **child flow** that retries independently keeps the parent
  flow's run history cleaner and can simplify retry logic — but child
  flows are a premium feature (licensing consideration — cross-reference
  the Power Apps skill's `licensing-and-limits.md`, since the same
  premium/standard connector economics apply to premium flow features).
- **Scope:** Situational alternatives — the skill should default to
  recommending #2 (copy-first) but mention these as options when #2's
  restructuring isn't feasible for the flow's actual shape.
- **Live-search trigger:** Medium — the HTTP bypass trick depends on a
  specific SharePoint REST endpoint that could change; the check-out/
  discard-checkout workaround is community-reported and worth
  re-verifying.

## 4. False positive: identical error message, unrelated root cause
- **Default habit:** Seeing a "file is locked" or "unable to acquire a
  lock" error and assuming it's always the session-lingering issue above,
  applying a delay/retry fix without further diagnosis.
- **Why this matters:** A practitioner-reported case hit a **very similar
  but distinct error** — "Graph API is unable to acquire or refresh a lock
  on the file because it is already locked" (HTTP 409) — when adding rows
  to an Excel table, where the *actual* root cause was that the input data
  included **fields that no longer existed in the Excel table's current
  schema** (the table had been restructured after the flow was built). No
  amount of delay or retry fixed this, because the file wasn't
  meaningfully "locked" in the sense this file otherwise describes — the
  error message was misleading. A separate contributing factor reported in
  the same investigation: referencing the Excel file by **Id** rather than
  by **Path** in the connector's File parameter also correlated with
  intermittent instances of this specific error for some users, though the
  mechanism for why isn't fully confirmed.
- **Correct diagnostic approach:** before applying a lock-focused fix,
  verify:
  - Does the data being written actually match the current table schema
    (no removed/renamed columns being referenced)?
  - Is the file referenced consistently the same way (by Path vs. by Id)
    throughout the flow, especially after any action that returns a
    file/table identifier used downstream?
  - Only after ruling these out, treat it as a genuine session-lock issue
    and apply the patterns in #2/#3.
- **Scope:** Excel table-writing actions specifically (Add a row, Update a
  row) — this is a narrower, distinct failure mode from the broader
  read-then-move lock issue in #1.
- **Severity trigger:** Any Excel table write action failing with a lock-
  sounding error after the table structure has changed, or after a flow
  was recently modified to reference the file differently.
- **Live-search trigger:** Medium-high — this is a narrower, less-
  documented community-reported pattern; treat as a diagnostic hypothesis
  to check, not a confirmed universal cause.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Identified whether a flow reads/writes an Excel or Word file and
      then performs a file-management operation (Move/Delete/Copy-
      overwrite/Update properties) on that same file later in the run
- [ ] Confirmed which Excel connector variant is in use (Business vs.
      OneDrive) if the exact lock duration matters
- [ ] Default fix applied: copy file to staging, read/write the copy, only
      then touch the original — rather than a fixed delay or blind retry
      loop against the original file
- [ ] Retry policy added on the copy/delete staging steps themselves, for
      the separate short-lived concurrent-run lock scenario
- [ ] For a genuine need to delete a locked file specifically: bypass-
      shared-lock HTTP technique considered as an option
- [ ] For any "locked"-sounding error on an Excel table write action:
      ruled out schema mismatch and file-reference-by-Id-vs-Path issues
      before assuming it's a session lock
- [ ] If child flows are used for retry isolation: licensing implications
      checked (premium feature)
