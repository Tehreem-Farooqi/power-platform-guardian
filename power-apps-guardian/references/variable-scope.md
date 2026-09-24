# Power Apps Variable Scope Reference

## Core principle
Power Apps has three variable scopes. The guiding rule across all of them:
**pick the smallest scope that satisfies the requirement.** Defaulting to
global variables for everything is the most common AI-written/naive habit,
and it isn't just a style issue — it has real memory, maintainability, and
regression-risk costs.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written without this knowledge |
| Why it matters | The real cost — memory, regression risk, correctness |
| Correct pattern | The fix, with real code |
| Scope | Where this applies |
| Severity trigger | When this stops being optional |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## The three scopes at a glance

| Scope | Function | Lifetime | Accessible from |
|---|---|---|---|
| Global | `Set(varName, value)` | Until app session ends or cleared | Any screen, whole app |
| Context/local | `UpdateContext({varName: value})` | Until user leaves that screen | Only the screen it was set on |
| Block/one-time | `With({varName: value}, formula)` | Only within that single formula | Only inside the `With()` call |

---

## 1. Defaulting to global variables for everything
- **Default habit:** Using `Set()` for state that's only ever needed on one
  screen, "just in case" it's needed elsewhere later.
- **Why it matters:** Global variables are accessible and modifiable from
  anywhere in the app. That convenience is also the cost: it increases the
  risk of regressions during refactoring (anything, anywhere, can be
  changing the same value) and makes the app harder to hand off to another
  developer, since scope no longer tells you anything about where a value
  is actually used. It's also a real memory cost — context variables are
  cleared when the user leaves the screen; globals persist for the whole
  session regardless of whether they're still needed.
- **Correct pattern:** Use global variables only when app-wide state is
  genuinely required — e.g. the signed-in user's role, a shopping cart, a
  side-menu open/closed toggle used across multiple screens. Use context
  variables (`UpdateContext`) for anything screen-local: form input state,
  a loading spinner's visibility, a screen-specific counter.
  ```
  // Global — needed everywhere
  Set(gblUserRole, "Admin");

  // Context — needed only on this screen
  UpdateContext({ locShowLoader: true });
  ```
- **Scope:** Universal.
- **Severity trigger:** Any app with more than a couple of screens — the
  regression risk compounds as the app grows.
- **Live-search trigger:** Low.

## 2. Not using `With()` for one-time/intermediate calculations
- **Default habit:** Creating a global or context variable purely to hold
  an intermediate calculation used once within a single formula, then never
  referenced again.
- **Why it matters:** This pollutes the app's variable namespace with
  values that don't need to persist anywhere, making the app harder to read
  and reason about, and carries an unnecessary (if small) memory cost
  compared to a value that only exists for the duration of one formula
  evaluation.
- **Correct pattern:** Use `With()` for values needed only within a single
  formula:
  ```
  With(
      { totalPrice: Sum(colCart, Price * Quantity) },
      If(totalPrice > 100, "Free shipping", "Shipping: $" & (totalPrice * 0.1))
  )
  ```
- **Scope:** Universal.
- **Severity trigger:** Any formula doing a calculation that's reused
  multiple times within itself, or that would otherwise "leak" into a
  Set/UpdateContext unnecessarily.
- **Live-search trigger:** Low.

## 3. Inline `User()` calls instead of caching at OnStart
- **Default habit:** Calling `User().Email` or `User().FullName` directly
  inline, repeatedly, throughout the app's formulas.
- **Why it matters:** This is a performance issue disguised as a scope
  issue. Each inline `User()` call queries the data source again. Even
  storing the user's info in a **context** variable is measurably less
  performant than a global variable, because a context variable set per
  screen still means re-querying each time that screen opens — only a
  global variable set once at `OnStart` avoids repeat queries entirely for
  the whole session.
- **Correct pattern:**
  ```
  // App.OnStart
  Set(gblUserEmail, User().Email);
  Set(gblUserFullName, User().FullName);
  ```
  Then reference `gblUserEmail`/`gblUserFullName` everywhere instead of
  calling `User()` inline.
- **Scope:** Universal — cross-reference `performance.md` #6 (OnStart
  guidance) for the balance between "cache this at OnStart" and "don't
  overload OnStart" — user info caching is specifically called out as
  justified OnStart work, unlike heavy data loading which should generally
  move to OnVisible.
- **Severity trigger:** Any app referencing `User()` more than once or twice.
- **Live-search trigger:** Low.

## 4. Custom component variable scope — a distinct, less-obvious mechanism
- **Default habit:** Assuming variables inside a custom (canvas) component
  behave the same as variables in the main app, or trying to use
  `UpdateContext` inside a component.
- **Why it matters:** `UpdateContext` **cannot be used inside custom
  components at all**. Components use `Set()`/`Collect()` only, and their
  actual scope depends on a component-level toggle:
  - **"Access app scope" OFF (default):** variables declared with `Set()`
    inside the component are scoped **only to that component instance**.
    If the same component is used multiple times on a screen or app, each
    instance holds its own independent copy — they do not share state.
  - **"Access app scope" ON:** variables declared with `Set()` inside the
    component behave as true **global** variables — shared across the
    whole app, including all instances of that component and the main app
    itself. This also means if the same component is deployed more than
    once, other instances will be affected by changes.
- **Correct pattern:** Deliberately choose the toggle state based on
  whether component instances should be independent (off, default — safer,
  avoids surprising cross-instance interference) or intentionally shared
  (on — only when genuinely needed for app-wide coordination).
- **Scope:** Custom/canvas components specifically.
- **Severity trigger:** Any app using custom components with internal state
  — check this toggle explicitly rather than assuming default behavior,
  since getting it backwards causes either unwanted state-sharing bugs or
  unexpected instance isolation.
- **Live-search trigger:** Medium — component framework behavior is an area
  Microsoft continues to develop; worth a periodic check if working with
  newer component types (code components / PCF vs. classic canvas
  components may differ).

## 5. Naming convention (maintainability, not a bug — but AI-written code
   consistently skips this)
- **Default habit:** No consistent prefix distinguishing variable scope at
  a glance (`userRole`, `showLoader`, `count` — impossible to tell scope
  from the name alone).
- **Why it matters:** Without a convention, a developer reading a formula
  has to go find where a variable was declared to know whether changing it
  will affect the whole app or just the current screen — a real source of
  accidental cross-screen bugs during maintenance.
- **Correct pattern:** A common, widely-recommended convention:
  - `gbl` prefix for global variables (`gblUserRole`)
  - `loc` prefix for context/local variables (`locShowLoader`)
  - `col` prefix for collections (`colCartItems`)
  This isn't an official Microsoft-enforced standard, but it's a
  widely-adopted community convention worth applying consistently in any
  code this skill generates.
- **Scope:** Universal, convention-level.
- **Severity trigger:** Worth applying by default in all generated code,
  regardless of app size.
- **Live-search trigger:** Low.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Used the smallest scope that satisfies the requirement (With > Context
      > Global, in order of preference where each is sufficient)
- [ ] No `User()` calls left inline/repeated — cached once in a global
      variable at OnStart
- [ ] Custom component "Access app scope" deliberately set, not left at
      default without consideration
- [ ] `UpdateContext` not used inside a custom component (not possible;
      `Set()` used instead with scope toggle considered)
- [ ] Consistent `gbl`/`loc`/`col` naming prefix applied
