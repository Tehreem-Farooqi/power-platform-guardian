---
name: power-apps-guardian
description: Helps write Power Apps canvas app formulas and app structure that avoid the failures that only show up after deployment or at scale — delegation, memory/performance, licensing surprises, silent error handling gaps, column-type mismatches, variable scope bugs, offline sync issues, ALM/environment drift, component reuse traps, and accessibility gaps. Use whenever writing or reviewing Power Fx formulas, designing data sources, building galleries, patching/updating records, planning offline capability, structuring solutions for deployment, or building reusable components in a Power Apps canvas app — especially against SharePoint, Dataverse, SQL Server, Salesforce, Excel/OneDrive, or Dynamics 365. Checks work against ten reference files before finalizing, follows a verification protocol for anything that may have changed since research was done, and documents what was caught for future reference.
---

# Power Apps Guardian

**Power Apps side: complete (10 reference files).** Power Automate is
intentionally out of scope for this skill — planned as a separate,
parallel skill later. This skill does not cover every conceivable Power
Apps topic (e.g. AI Builder integration, portals/Power Pages specifics,
deep PCF/code-component development are not covered) — treat it as
thorough on the categories listed below, not universally exhaustive.

## Read this first, every time: the verification protocol

Before using anything else in this skill, read
`references/verification-protocol.md`. In short: several categories here
are things Microsoft actively changes (pricing, quotas, plan names,
connector classifications, offline feature limitations). Each reference
file marks its own volatility. **Never state a specific number, price,
quota, or availability claim with confidence without either verifying it
live first, or explicitly flagging it as potentially stale.** This is the
difference between a skill that helps and one that confidently repeats
outdated information — treat it as a hard requirement, not a suggestion.

## What this skill does

When writing or reviewing Power Fx formulas or Power Apps structure, this
skill checks work against these reference files before finalizing:

1. `references/delegation.md` — connector-by-connector delegation rules,
   known traps, column-type exceptions, and fixes. Covers Dataverse, SQL
   Server/Azure SQL, SharePoint, Salesforce, Dynamics 365, Common Data
   Service (legacy, flagged deprecated), Excel/OneDrive, Azure Table
   Storage.
2. `references/performance.md` — memory/performance anti-patterns
   (Batch Patch vs. ForAll+Patch, N+1 queries, OnStart bloat, oversized
   collections, control count, connection limits), why they break, and the
   correct pattern to use instead.
3. `references/licensing-and-limits.md` — standard vs. premium connector
   licensing traps, current plan structure, Dataverse capacity cost, API
   quota risk. **High volatility across the board — always re-verify
   current state live before giving licensing advice.**
4. `references/error-handling.md` — IfError/OnError patterns, Patch
   failure handling, form OnSuccess/OnFailure, batch-Patch error handling,
   input validation, error logging, and a summary checklist.
5. `references/column-data-type-quirks.md` — SharePoint internal vs.
   display names, Choice/Lookup/Person column comparison rules (Record
   types, not scalars), and why a perfectly delegable, performant formula
   can still be wrong.
6. `references/variable-scope.md` — global vs. context vs. `With()`
   scoping, the User()-caching pattern, custom component scope rules
   ("Access app scope" toggle), and naming conventions.
7. `references/offline-and-mobile-sync.md` — Dataverse built-in
   offline-first vs. manual SaveData/LoadData, column-level conflict
   resolution mechanics, unsupported offline functions, sync-error
   handling, and the change-queue pattern. **Medium-high volatility —
   verify current capability status live.**
8. `references/alm-and-environments.md` — environment variables and
   connection references, managed vs. unmanaged solutions, unmanaged-layer
   drift risk, solution structuring, reference/seed data deployment, and
   secrets handling.
9. `references/component-reuse.md` — Components vs. Component Libraries,
   the silent-local-copy trap when editing library components in-app, the
   property-loss-on-update bug pattern, and nested-component performance.
10. `references/accessibility.md` — screen name/reading order, TabIndex
    (including the retired custom-index behavior), AccessibleLabel,
    contrast, live regions, heading roles, and the built-in Accessibility
    Checker. Note: often a legal/procurement requirement, not just UX polish.

## How to use this skill

1. **Identify the data source(s) involved first.** Delegation and
   performance guidance is source-specific — always check which connector
   a given formula actually touches (see delegation.md's "common combo
   pattern" note; apps often mix sources).
2. **Filters/sorts/searches/aggregates against a cloud source** → check
   `delegation.md`. Flag non-delegable functions, compound-formula issues
   (aggregate-of-filter, nested `If`), column-type exceptions. Check actual/
   expected record count to judge real severity, not just the warning
   triangle's presence.
3. **Bulk updates, data loading, gallery design, OnStart/OnVisible logic**
   → check `performance.md`. Default to Batch Patch over ForAll+Patch for
   bulk updates — the single most common, costly AI-written anti-pattern
   — *except* in an offline sync-reconciliation loop, where per-record
   error isolation is deliberately preferred (see offline-and-mobile-
   sync.md's cross-reference note).
4. **Recommending a premium connector or Dataverse** (including for
   delegation reasons) → check `licensing-and-limits.md` and mention the
   licensing/cost consequence alongside the technical recommendation.
5. **Any formula that writes data (Patch, Collect, SubmitForm) or that
   could predictably fail** (LookUp on a maybe-missing record, division,
   a Flow call) → check `error-handling.md`'s summary checklist. An
   unwrapped Patch is as high-priority a catch as a ForAll+Patch
   performance issue.
6. **Any formula filtering, comparing, or displaying a SharePoint Choice,
   Lookup, or Person/Group column** → check `column-data-type-quirks.md`.
   These are Record types, not scalars.
7. **Any Set/UpdateContext/With usage, or a custom component with internal
   state** → check `variable-scope.md`. Default to the smallest sufficient
   scope; confirm no `User()` calls are left inline instead of cached once
   at OnStart.
8. **Any app with stated or implied offline requirements** (field use,
   spotty connectivity, "works without internet") → check
   `offline-and-mobile-sync.md` *first*, before writing other formulas for
   that app — the route choice shapes nearly everything else.
9. **Anything touching deployment, environments, or connections** →
   check `alm-and-environments.md`. Flag hardcoded environment-specific
   values immediately; flag direct edits to test/production environments.
10. **Building or reusing a Component or Component Library** → check
    `component-reuse.md`, especially before editing a library-sourced
    component directly inside a consuming app, and before/after any
    library update that changes a component's custom properties.
11. **Before considering any app "done"** → check `accessibility.md`'s
    summary checklist and recommend running the built-in Accessibility
    Checker, regardless of whether accessibility was explicitly requested.
12. **At the end of the working session, produce a short changelog** (see
    below) documenting what was caught, why, and how it was fixed.

## End-of-session documentation format

At the end of a working session where this skill caught and fixed real
issues, summarize in this structure:

```
### Power Apps Guardian — session notes

| Issue | Data source | Root cause | Fix applied | Reusable snippet? |
|---|---|---|---|---|
```

Keep entries terse — one row per distinct issue caught, not per formula
touched. This is meant to be skimmed by a future developer, not read as a
narrative.

## Reference files

- `references/verification-protocol.md` — **read first, every session.**
  How to handle volatile information, confidence levels, and when to
  search live instead of trusting cached content.
- `references/delegation.md` — read when a formula touches Filter, Search,
  LookUp, Sort, SortByColumns, or any aggregate against a cloud data source.
- `references/performance.md` — read when writing bulk updates
  (ForAll/Patch), OnStart/OnVisible logic, gallery structure, or collection
  handling.
- `references/licensing-and-limits.md` — read when choosing/recommending a
  data source or connector, or when the user asks about cost, licensing, or
  rollout to a user base. Always cross-check live before quoting numbers.
- `references/error-handling.md` — read when writing any Patch, Collect,
  SubmitForm, or other formula that writes data or could predictably fail.
  Apply its summary checklist before calling a formula finished.
- `references/column-data-type-quirks.md` — read when a formula touches a
  SharePoint Choice, Lookup, or Person/Group column, or when columns were
  renamed after list creation.
- `references/variable-scope.md` — read when writing Set/UpdateContext/With
  logic, caching User() info, or building custom components with internal
  state.
- `references/offline-and-mobile-sync.md` — read as soon as offline or
  spotty-connectivity requirements are mentioned, before other formulas for
  that app are written.
- `references/alm-and-environments.md` — read when a formula/app references
  an external URL, connection, or credential, or when deployment/environment
  promotion is discussed.
- `references/component-reuse.md` — read when building, updating, or
  consuming a Component or Component Library.
- `references/accessibility.md` — read before finalizing any screen's
  layout, and always as part of a pre-completion review pass.
