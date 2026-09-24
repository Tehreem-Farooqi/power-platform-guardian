# Power Apps Column & Data-Type Modeling Quirks

This file covers **why formulas break or behave confusingly around specific
column types and naming**, distinct from `delegation.md` (which covers
*whether a query pushes server-side*) and `performance.md` (*speed/memory*).
A formula can be perfectly delegable and performant and still be wrong or
broken because of a column-type mismatch — this file is that third axis.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written without this knowledge |
| Why it breaks / confuses | The actual mechanism |
| Correct pattern | The fix, with real code |
| Scope | Which connector(s) this applies to |
| Severity trigger | When this actually bites |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. SharePoint internal names vs. display names
- **Default habit:** Writing formulas that assume a SharePoint column's
  Power Fx reference matches whatever it's currently labeled in the list UI.
- **Why it breaks:** Every SharePoint column has **two names**: an internal
  name (fixed permanently at creation) and a display name (editable anytime,
  purely cosmetic). Power Fx formulas reference columns by **internal name**
  in most contexts, but labels/cards in a generated form may show the
  **display name**. Renaming a column in SharePoint changes only the display
  name — the internal name, and therefore the working formula reference,
  never changes. This is deliberate: it's what prevents existing formulas
  from breaking when someone renames a column. But it causes two very
  different real problems:
  1. **Confusion after a rename:** a dev renames "MyColumn1" to
     "YourColumn1" in SharePoint, then can't find "YourColumn1" anywhere in
     their formulas — because Power Apps still shows/uses "MyColumn1"
     internally. Nothing is actually broken; the dev just doesn't know where
     to look.
  2. **Genuinely opaque internal names:** when a SharePoint list is created
     by importing from Excel, or under certain other creation paths, the
     internal names can come out as meaningless strings like `field_2` or
     random-looking identifiers — completely disconnected from the display
     name shown in the list. This is reported as confirmed platform
     behavior (Microsoft has acknowledged work-in-progress on this), not
     user error.
  3. **The Title column special case:** the first/default column is always
     referenced as `Title` in Power Fx regardless of what it's displayed as
     in SharePoint — a very common point of confusion since "Title" rarely
     matches what the column is actually being used for (e.g. a list where
     the "real" title column has been renamed to "Need" or "Name").
- **Fix / mitigation:**
  - To find a column's true internal name: SharePoint list settings >
    Columns > click the column > the internal name appears in the page URL
    after `Field=`.
  - Don't assume a rename in SharePoint requires any app changes — the
    working formula reference doesn't change. But DO expect to need to
    manually update **display labels** in generated forms/cards, since
    those don't auto-sync to a renamed display name in all cases.
  - When building fresh, consider setting internal names deliberately
    correct at column-creation time (before first save) rather than
    relying on renames later, especially for lists created via Excel import.
  - Special characters and spaces in a display name get encoded into the
    internal name (e.g. spaces become `_x0020_`) — expect this in older
    apps' code; current Power Apps versions (post ~3.24042) use plain
    display-name syntax with quotes for spaced names rather than requiring
    manual `_x0020_` encoding, but be aware both forms may appear in
    existing/legacy code.
- **Scope:** SharePoint-specific. Not an issue for Dataverse (logical names
  are far more stable/predictable) or SQL Server.
- **Severity trigger:** Any SharePoint list where columns were renamed after
  creation, or created via Excel import — check this early rather than
  after formulas mysteriously don't work.
- **Live-search trigger:** Low-medium — the mechanism is stable, but the
  "random opaque internal name" bug behavior on Excel-imported lists has an
  open Microsoft acknowledgment of in-progress work, worth a periodic check.

## 2. Choice columns — `.Value` vs. bare comparison
- **Default habit:** Comparing a SharePoint Choice column directly to a
  plain text string, or filtering/searching it like a plain text column.
- **Why it breaks:** A Choice column value isn't a plain string in Power
  Fx — it's a complex type. Comparing or filtering it directly against text
  produces type errors or silently fails to match. It also **isn't
  delegable as a complex field type** (cross-reference `delegation.md`) —
  complex types (choice, lookup, date, metadata, person/group) aren't
  delegable in SharePoint, so filtering/searching them pulls the whole list
  locally past the row limit.
- **Correct pattern:**
  ```
  // Filtering/searching a Choice column — use .Value
  Filter('List', StartsWith(Locations.Value, TextSearchBox1.Text))

  // Populating a dropdown of all choice options for a column
  Choices(ParkingMasterGarages.Area)

  // Setting a default selected value from a choice column
  ThisItem.Area.Value
  ```
- **Multi-select choice columns:** these use a **Choices** column type, not
  Lookup — the resulting table has only one field, `Value`, per selection
  (no ID needed since it's not referencing another list). To list selected
  values as text:
  ```
  Concat(YourComboBoxName.SelectedItems, Value & ", ")
  ```
- **Scope:** SharePoint-specific (Choice column type doesn't exist the same
  way in Dataverse — Dataverse uses Choice/Option Set columns with somewhat
  different but analogous `.Value` handling; verify live if working against
  Dataverse choice columns specifically).
- **Severity trigger:** Any SharePoint list with Choice columns used in
  search, filter, or comparison logic.
- **Live-search trigger:** Low.

## 3. Lookup columns — Record type, not scalar
- **Default habit:** Comparing a SharePoint Lookup column directly against
  a plain value (text or number), expecting it to behave like a foreign key.
- **Why it breaks:** A Lookup column returns a **Record** (with at minimum
  `.Id` and `.Value` sub-fields), not a scalar. Comparing a Record to Text or
  Number produces "incompatible types for comparison" errors — this is one
  of the most consistently reported SharePoint+Power Apps error messages.
- **Common wrong attempts and why they fail:**
  ```
  // WRONG — comparing Record to Text
  Filter(Suggestions, Mailbox = MailboxChoice1.Selected.Mailbox)
  // Error: "You can't compare types: Record, Text"

  // WRONG — comparing by .Id when the other side is a filtered table, not
  // matching type, or ID mismatch between environments/list instances
  Filter(Questions, Adress.Id = varSelectedAdress.ID)
  // May silently return 0 items even when the ID "looks" correct — ID
  // matching across lookup relationships is fragile if the comparison
  // value isn't sourced from the same lookup relationship correctly.
  ```
- **Correct pattern:**
  ```
  // Compare .Value to a text value
  Filter(Suggestions, Mailbox.Value = MailboxChoice1.Selected.Mailbox)
  ```
- **Known fragility:** filtering by `.Id` instead of `.Value` on a Lookup
  column is reported as unreliable/inconsistent in some scenarios even when
  the ID is confirmed correct via a test label — prefer `.Value` comparisons
  where uniqueness allows it, but be aware `.Value` comparisons risk
  **false matches on duplicate display text** if the underlying list allows
  duplicate values. There's a genuine tradeoff here with no universally
  "safe" default — flag this explicitly rather than picking one silently.
- **Practitioner consensus (strong, repeated across sources):** SharePoint
  Lookup columns are widely considered a persistent source of friction.
  Community guidance often recommends **avoiding SharePoint Lookup columns
  where possible** in favor of simpler column types, or migrating to
  Dataverse (which models relationships far more cleanly) once an app's
  data model needs real relational integrity.
- **Scope:** SharePoint-specific. Dataverse lookups behave more predictably
  (proper relational foreign keys) — flag this SharePoint-specific fragility
  explicitly rather than assuming it generalizes to Dataverse.
- **Severity trigger:** Any SharePoint list with Lookup columns used in
  filter/comparison logic — check early, this is a near-certain point of
  confusion if not addressed proactively.
- **Live-search trigger:** Low — this is long-standing, consistently
  reported platform behavior, not something likely to change soon.

## 4. Person/Group columns (cross-reference)
- Covered in `delegation.md` under SharePoint column-type exceptions:
  only `.Email` and `.DisplayName` sub-fields delegate. Same Record-type
  principle as Lookup columns applies — a Person field is a Record, not a
  scalar, so direct comparison against plain text will fail the same way
  Lookup comparisons do. Use `.Email` or `.DisplayName` explicitly.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Confirmed which name (internal vs. display) a SharePoint formula
      reference actually needs, especially after any column rename
- [ ] Checked whether the list was created via Excel import (higher risk of
      opaque/random internal names)
- [ ] Choice column comparisons use `.Value`, not bare comparison
- [ ] Lookup column comparisons use `.Value` or `.Id` deliberately, with the
      duplicate-value risk considered for `.Value` comparisons
- [ ] Person/Group column comparisons use `.Email` or `.DisplayName`
      explicitly, not bare comparison
- [ ] For any SharePoint list with heavy Lookup column usage, flagged
      whether Dataverse would be a better-fit data model going forward
