# Power Apps Error Handling Reference

## Prerequisite — often missed entirely
`IfError`, `IsError`, and the app's `OnError` property require **Formula-level
error management** to be enabled first: App settings > Upcoming features >
"Error handling at the formula level" (naming may vary by release — verify
in current Studio). Without this enabled, these functions aren't fully
available. Check for this before assuming any error-handling code will work.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written without this knowledge |
| Why it matters | The real-world failure mode this prevents |
| Correct pattern | The fix, with real code |
| Scope | Universal / data-source-specific |
| Severity trigger | When this stops being optional |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. Unwrapped Patch/Collect calls
- **Default habit:** Bare `Patch(...)` or `Collect(...)` with no error
  handling — the AI-written formula assumes the write always succeeds.
- **Why it matters:** Network connectivity, data source availability, and
  user permissions can never be safely assumed, even for a well-formed
  record. An unwrapped Patch can fail silently — the user thinks they saved
  their data and they didn't, with zero indication anything went wrong.
  Not needed for local collections in memory only.
- **Correct pattern:**
  ```
  IfError(
      Patch(
          'Invoices',
          Defaults('Invoices'),
          {
              CustomerNumber: "C0001023",
              InvoiceDate: Date(2022, 6, 13),
              PaymentTerms: "Cash On Delivery",
              TotalAmount: 13423.75
          }
      ),
      // On failure
      Notify("Error: the invoice could not be created", NotificationType.Error),
      // On success
      Navigate('Success Screen')
  )
  ```
- **Scope:** Universal — applies to any write against any cloud data source.
- **Severity trigger:** Every single Patch/Collect against a live data
  source, no exceptions. This is a baseline, not an edge-case precaution.
- **Live-search trigger:** Low.
- **Skill priority:** HIGH — alongside Batch Patch, this is one of the most
  consistently missing patterns in AI-generated Power Fx.

## 2. Generic, unhelpful error messages
- **Default habit:** `Notify("Error", NotificationType.Error)` or similarly
  vague messaging that tells the user nothing actionable.
- **Why it matters:** A user who sees "Error" with no context can't tell
  whether to retry, contact support, check their connection, or that their
  data is simply gone.
- **Correct pattern:** Use `FirstError.Message` for the underlying detail,
  and/or `FirstError.Kind` to branch on error type for a more human message:
  ```
  IfError(
      Patch(Tasks, Defaults(Tasks), {Title: TextInput1.Text}),
      Notify(
          Switch(
              FirstError.Kind,
              ErrorKind.Network, "Connection problem — check your internet and try again.",
              ErrorKind.Conflict, "Someone else edited this record — refresh and try again.",
              ErrorKind.Permission, "You don't have permission to save this record.",
              "Something went wrong. Please try again or contact support. Details: " & FirstError.Message
          ),
          NotificationType.Error
      )
  )
  ```
- **Common `ErrorKind` values encountered against SharePoint/Dataverse:**
  `Network` (connectivity/data source unreachable), `Conflict` ("Conflicts
  exist with changes on the server" — another user/process modified the
  record between read and write), `Permission` (user lacks create/modify
  rights), `Unknown` (Message field usually carries the connector's raw
  error text in this case).
- **Conflict-specific fix:** Call `Refresh()` on the data source, then retry
  the operation — don't just show an error and stop.
- **Scope:** Universal.
- **Severity trigger:** Any user-facing save/submit action.
- **Live-search trigger:** Low.

## 3. Forms: missing OnSuccess/OnFailure handling
- **Default habit:** `SubmitForm(frm_Invoice)` called with no follow-up
  logic, or code written directly after the SubmitForm call expecting it to
  run synchronously after success.
- **Why it matters:** SubmitForm is asynchronous — code placed directly
  after it in the same formula does NOT wait for success/failure. Logic
  needs to live in the form's own `OnSuccess`/`OnFailure` properties.
- **Correct pattern:**
  ```
  // OnSelect of the submit button
  SubmitForm(frm_Invoice);

  // OnSuccess property of the form
  Navigate('Success Screen');

  // OnFailure property of the form
  Notify("Error: the invoice could not be created", NotificationType.Error);
  ```
- **Scope:** Universal — applies to any Form control submission.
- **Severity trigger:** Every form using SubmitForm.
- **Live-search trigger:** Low.

## 4. No global catch-all for unexpected errors
- **Default habit:** Only wrapping the "obvious" risky calls (Patch,
  SubmitForm) in IfError, with nothing catching truly unexpected failures
  elsewhere in the app.
- **Why it matters:** IfError only catches errors at the specific point it
  wraps. Anything outside that — an unexpected runtime error somewhere else
  in the app — surfaces as a raw, unstyled error bar the user doesn't
  understand.
- **Correct pattern:** Set the app's `OnError` property as a safety net:
  ```
  // App.OnError
  Notify("Something went wrong: " & FirstError.Message, NotificationType.Error)
  ```
- **Critical interaction to know:** **If IfError handles an error,
  App.OnError does NOT fire for that same error.** IfError marks it as
  handled, so the global handler never sees it — no risk of a double
  notification. Use IfError for operations you know might fail (Patch,
  running a Power Automate flow, LookUp where the record might not exist,
  division where the denominator could be zero); use OnError purely as the
  net for everything you didn't anticipate.
- **Scope:** Universal — one OnError per app, but the pattern (local
  handling + global net) applies everywhere.
- **Severity trigger:** Every production app should have this set, even a
  small one — costs nothing and prevents raw error bars reaching users.
- **Live-search trigger:** Low.

## 5. Bulk/batch Patch error handling
- **Default habit:** Nesting multiple Patch calls (e.g. a main record plus
  detail records) without a unified error-handling wrapper — one failure
  partway through leaves the app in an inconsistent, hard-to-diagnose state.
- **Why it matters:** With a true batch Patch (see `performance.md` #1 —
  `Patch(Datasource, CollectionOfChanges)`), some records in the batch can
  fail while others succeed. The dev needs to know which failed, not just
  that "something" failed.
- **Correct pattern:** Wrap the whole batch Patch in one `IfError` at the
  top level rather than wrapping each individual Patch separately:
  ```
  IfError(
      Patch(IceCream, baseRecords, changeRecords),
      Notify("Some records failed to update: " & FirstError.Message, NotificationType.Error)
  )
  ```
- **Scope:** Universal, specifically relevant wherever Batch Patch (from
  performance.md) is used.
- **Severity trigger:** Any bulk update where partial failure is possible
  and would leave data in an inconsistent state if unnoticed.
- **Live-search trigger:** Low.

## 6. No input validation before submit
- **Default habit:** Submitting user input directly to Patch/SubmitForm
  without checking it's well-formed first, relying entirely on the data
  source to reject bad data (if it does at all).
- **Why it matters:** Server-side rejection produces a worse user experience
  than catching the problem before the round-trip — and some invalid states
  (e.g. a required field silently defaulting to blank) may not error at all,
  just save wrong data.
- **Correct pattern:** Validate inputs (required fields non-blank, numeric
  fields actually numeric via `IsNumeric`/`Value` with IfError fallback,
  etc.) before calling Patch/SubmitForm, and give immediate inline feedback
  rather than waiting for a round-trip failure.
- **Scope:** Universal.
- **Severity trigger:** Any user-entry form.
- **Live-search trigger:** Low.

## 7. No error logging/traceability
- **Default habit:** Notify-and-forget — the user sees an error message,
  but nothing is recorded for the developer to diagnose later.
- **Why it matters:** Without a trace, a recurring or hard-to-reproduce
  error is invisible to the dev team until a user complains — and even
  then, there's no detail beyond what the user remembers.
- **Correct pattern:** Use the `Trace()` function to log details visible in
  Monitor, and/or patch a dedicated errors-log table, alongside the
  user-facing Notify:
  ```
  IfError(
      Patch(Characters, Defaults(Characters), {Allegiance: "Dark Side"}),
      Notify("Oops, something went wrong", NotificationType.Error, 2000);
      Trace("Patch failed: " & FirstError.Message, TraceSeverity.Error, {
          ErrorSource: FirstError.Source,
          UserEmail: User().Email,
          OccurredAt: Now()
      })
  )
  ```
- **Scope:** Universal, more valuable as app complexity/user count grows.
- **Severity trigger:** Recommended for any production app; near-essential
  once multiple developers/support staff need to diagnose issues after the
  fact.
- **Live-search trigger:** Low.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Formula-level error management enabled in app settings
- [ ] Every Patch/Collect against a live data source wrapped in IfError
- [ ] Error messages branch on FirstError.Kind where reasonable, not generic
- [ ] Forms use OnSuccess/OnFailure, not code chained after SubmitForm
- [ ] App.OnError set as a global safety net (understanding it won't
      double-fire on already-handled IfError errors)
- [ ] Batch Patch wrapped once at the top level, not per-record
- [ ] Input validated before submission, not only relying on server rejection
- [ ] Errors logged via Trace() or an error table for production apps
