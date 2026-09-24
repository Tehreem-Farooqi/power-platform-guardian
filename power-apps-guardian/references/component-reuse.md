# Power Apps Component Reuse Reference

## Core distinction — two different things called "components"
- **Canvas components** (this file's focus): reusable building blocks
  within/across canvas apps, built with Power Fx, custom properties, and
  standard controls.
- **Code components (Power Apps Component Framework / PCF):** a different,
  more advanced system requiring actual code, used when device features
  (camera, microphone) or framework-level capabilities are needed. Not
  covered in depth here — flag as a distinct, heavier-weight option if a
  request needs device-level access that canvas components can't provide.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written/recommended without this knowledge |
| Why it matters | The real cost or failure mode |
| Correct pattern | The fix / correct setup |
| Scope | Where this applies |
| Severity trigger | When this bites |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. Copy-pasting controls instead of componentizing
- **Default habit:** Rebuilding the same header, footer, or card layout
  manually on every screen, or copy-pasting a control group repeatedly.
- **Why it matters:** A later design change (e.g. "update our brand color")
  then requires manually updating every screen, every button, every label
  individually — slow and highly error-prone, guaranteed to produce
  inconsistency over time.
- **Correct pattern:** Componentize any UI pattern used more than once
  within an app. Updating the component definition once propagates to all
  instances in that app automatically. Componentizing also reduces
  duplication effort and can improve performance versus repeated separate
  control trees.
- **Scope:** Universal, within a single app.
- **Severity trigger:** Any UI element appearing on 2+ screens.
- **Live-search trigger:** Low.

## 2. Manual copy-paste across apps instead of a Component Library
- **Default habit:** Copying a component from one app to another manually
  when the same pattern (header, footer, standard form) is needed across
  multiple apps.
- **Why it matters:** Manual copying loses version control, makes tracking
  updates impossible, and produces inconsistent fixes across apps over
  time — a fix applied to one app's copy doesn't reach the others.
  Additionally, the older feature that allowed direct component
  import/export between individual apps (without a library) has been
  **disabled by default and is being phased out** — it is explicitly not
  the recommended path going forward.
- **Correct pattern:** Use a **Component Library** — a dedicated container
  for components meant to be reused across multiple apps. Libraries provide
  discoverability (search/find components), a formal update-notification
  flow to consuming apps, and dependency tracking.
- **Scope:** Cross-app reuse specifically (within-app reuse is Component,
  not Library — see #1).
- **Severity trigger:** Any pattern needed in 2+ separate apps, especially
  organization-standard elements (headers, footers, branded buttons).
- **Live-search trigger:** Low-medium — component library capabilities have
  evolved (e.g. "enhanced component properties" with functions/behavior
  functions has moved through preview/GA stages), worth checking current
  feature maturity for advanced property types.

## 3. Editing a library-sourced component directly inside the consuming app
- **Default habit:** Opening a component that was imported from a library,
  and directly editing it inside the app that's using it, rather than
  editing the source in the library.
- **Why it breaks:** You **cannot edit a library-imported component
  in-place** in the consuming app. Attempting to do so creates a **local
  copy** of the component inside that app — silently. That local copy then
  **stops receiving updates** when the library component is later changed,
  because the app is no longer referencing the shared definition at all.
  This is a documented, known-limitation behavior, not a bug — but it's an
  extremely easy trap to fall into without realizing what happened.
- **Correct pattern:** Always edit the component definition in the
  **Component Library** itself, then publish and let consuming apps review
  and accept the update. If an app-specific variation is genuinely needed,
  explicitly create a copy (an available, deliberate option) with full
  awareness that it will now diverge permanently from the shared source —
  don't let this happen accidentally via direct in-app editing.
- **Scope:** Any app consuming components from a shared library.
- **Severity trigger:** Any team with more than one component library
  consumer — this is a very common way for "the header component looks
  different in three of our five apps" bugs to originate.
- **Live-search trigger:** Low — documented, stable known-limitation
  behavior.

## 4. Adding new custom properties to a component and losing old ones on update
- **Default habit:** Adding a new input/output custom property to a
  component already in use across an app or multiple apps, then publishing
  the library update and accepting it in consuming apps without extra care.
- **Why it breaks:** This is a **reported, real-world bug pattern** (not
  purely theoretical): after accepting a library update that added new
  custom properties, existing property values already set on component
  instances can be **silently lost** — the new property appears, but a
  previously-set older property becomes unavailable/reset. This is
  distinct from the accidental-local-copy issue in #3; it can happen even
  when following the "correct" library-update workflow.
- **Correct pattern:**
  - Treat any component property change as a **breaking-change risk**, not
    a routine update — test the update in a non-production app copy first
    if the component is widely used.
  - After accepting a component library update anywhere it matters, review
    each instance's property panel rather than assuming values carried
    over correctly, especially for components with several custom
    properties already configured.
  - Document component property changes (even informally) so a dev
    reviewing a broken instance later knows a property was added/changed
    recently and can check for this specific issue first.
- **Scope:** Any component with custom properties, used in more than a
  trivial number of places.
- **Severity trigger:** Any component update that adds/removes/renames
  custom properties on a component with existing configured instances.
- **Live-search trigger:** Medium — this is reported community behavior
  rather than official documented behavior; verify current status live
  since Microsoft may have addressed it in platform updates since reports
  surfaced.

## 5. `UpdateContext` cannot be used inside components (cross-reference)
- Already covered in `variable-scope.md` #4, repeated here because it's
  specifically a component-reuse gotcha too: components use `Set()`/
  `Collect()` only, with behavior governed by the "Access app scope"
  toggle (off = per-instance isolated state, on = true global/shared
  state across all instances). Check this deliberately for any component
  holding internal state, especially before reusing that component across
  multiple apps where instance-isolation expectations may differ.

## 6. Nested components — real but under-documented performance cost
- **Default habit:** Nesting components several levels deep (a component
  containing another component containing another) without considering
  cost, treating it like ordinary container nesting.
- **Why it matters:** Cross-reference `performance.md` #7 — nested
  structures generally cost more than flat ones in Power Apps' internal
  complexity accounting, and this applies to nested components too. Unlike
  the flat control-count guidance, there isn't strong official public
  documentation quantifying nested-component cost specifically — flag this
  as worth watching rather than citing a specific number.
- **Scope:** Apps with deeply nested component hierarchies (3+ levels).
- **Severity trigger:** Noticeable app slowdown correlating with heavy
  nested-component usage — investigate flattening structure as a candidate
  fix.
- **Live-search trigger:** Medium-high — this is thin ground; verify with
  current sources rather than treating this file's note as authoritative.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Repeated UI patterns (2+ screens) turned into Components, not
      copy-pasted
- [ ] Cross-app reuse uses a Component Library, not manual copy or the
      deprecated import/export feature
- [ ] No direct in-app editing of library-sourced components — confirmed
      edits happen in the library source, or an app-specific copy was
      created deliberately, not accidentally
- [ ] Any component property change treated as a breaking-change risk;
      instance property panels reviewed after accepting a library update
- [ ] Component internal state ("Access app scope") deliberately set,
      cross-referencing `variable-scope.md` #4
- [ ] Deep component nesting (3+ levels) reviewed for performance impact
      if the app shows unexplained slowdown
