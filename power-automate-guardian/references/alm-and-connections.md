# Power Automate ALM & Connection References Reference

This is the flow-specific counterpart to the Power Apps skill's
`alm-and-environments.md`. The underlying mechanisms (environment
variables, connection references, managed/unmanaged solutions) are shared
platform concepts — read that file too for the general model. This file
covers what's specifically different or extra-risky for **flows**: who a
flow runs as, what happens when an owner leaves, and deployment gaps that
are easy to miss because a flow can look correctly deployed and still fail
the first time it actually runs.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written/recommended without this knowledge |
| Why it breaks | The actual mechanism |
| Correct pattern | The fix |
| Scope | Where this applies |
| Severity trigger | When this bites |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. Not understanding which identity a flow actually runs as
- **Default habit:** Assuming a flow always runs "as the person who built
  it," regardless of trigger type — leading to confusion when a flow
  behaves differently for different users, or fails for reasons that don't
  match the person who's currently looking at it.
- **Why it matters:** The identity a flow runs as **depends on the trigger
  type**, and getting this wrong leads to genuinely confusing debugging
  sessions (a flow that "works for me" but fails for a colleague, with no
  code difference):
  - **Manually started (instant) flows** run as **the user who starts the
    flow** — not the flow's owner/creator.
  - **Automated flows** (triggered by an event, e.g. "When an item is
    created") run as the user specified by the **connection**, which is
    typically the flow owner's connection unless a different connection is
    explicitly used. For certain Dataverse-triggered automated flows
    specifically, a **"Run as"** option exists on the trigger itself,
    letting you choose between: the **Flow owner**, the **Modifying
    user** (whoever's action caused the trigger to fire), or the **Row
    owner** (the Dataverse record's owner) — useful when the flow needs
    access rights the primary flow owner doesn't have, or vice versa.
  - **Scheduled flows** always run as the **flow's primary owner** — there
    is no "Run as" option and no way to change this, because there's no
    invoking user to model an option around (the trigger is time-based,
    not user-initiated).
- **Correct pattern:** Before debugging "works for one person, not
  another" behavior, identify the trigger type first — it determines
  whose permissions actually matter for that specific run. For Dataverse-
  triggered automated flows needing elevated or restricted access
  relative to the flow owner, use the "Run as" trigger setting
  deliberately rather than fighting permission issues with workarounds.
- **Scope:** Universal — applies to every flow, and is foundational to
  diagnosing a wide range of "works sometimes" bugs.
- **Severity trigger:** Any flow whose behavior seems inconsistent across
  different triggering users or contexts — check trigger type and
  run-as identity before assuming a logic bug.
- **Live-search trigger:** Low — this is stable platform architecture.

## 2. Multiple connection references where one would do, or vice versa
- **Default habit:** Either creating a new, separate connection reference
  for every flow/action without considering reuse, or conversely cramming
  unrelated services onto a single shared connection reference regardless
  of scope.
- **Why it matters:** A connection reference is a shell around an actual
  connection — actions in a flow use the reference, and the reference
  determines which real identity/credential is actually used at runtime.
  Unnecessary duplication makes environment promotion and connection
  auditing harder (more references to track, repoint, and audit per
  environment) with no real benefit.
- **Correct pattern:** Use a **single connection reference** per
  service/purpose unless there's a specific reason for more — the
  documented example of a legitimate reason being a flow that
  genuinely needs to connect into **multiple sites across multiple
  tenants**, where one reference per tenant/site is a real requirement,
  not duplication. Cross-reference the Power Apps skill's
  `alm-and-environments.md` #3 for the broader connection-reference
  structuring guidance (one-per-service-per-environment, centralized
  "Core Connections" solution pattern) — that guidance applies to flows
  identically.
- **Multi-developer coordination:** when more than one developer builds
  flows against the same services, establish a **leading/shared account**
  (or service account, see #4) up front and assign that single set of
  connection references consistently — retrofitting this after multiple
  developers have each created their own connections is significantly
  more painful than establishing the convention from the start.
- **Scope:** Any environment with more than one flow or more than one
  developer.
- **Severity trigger:** Any team with more than one developer, or any
  environment with more than a handful of flows.
- **Live-search trigger:** Low.

## 3. Flows breaking silently when an owner leaves, is deactivated, or
   changes role
- **Default habit:** A single named individual as the sole owner of a
  business-critical flow, with no co-owners and no plan for what happens
  when that person leaves or changes roles.
- **Why it breaks — three distinct, commonly-conflated failure modes:**
  - **Orphaned flow:** the flow has **no valid owner at all** (the
    creator/owner account was removed with no co-owner ever added). Only
    privileged admins can even see orphaned flows in the standard flow
    list — they don't show an owner in the Owners column, making them easy
    to miss during routine review. Fix requires an admin assigning a new
    co-owner via the Power Platform Admin Center (or via PowerShell/Power
    Apps admin cmdlets at bulk scale).
  - **Disabled/deactivated owner:** the account still technically exists
    as owner but is deactivated — the flow doesn't necessarily show as
    "orphaned" but **silently stops functioning** because the connection
    tied to that identity is no longer valid. This is easy to miss because
    the flow still *looks* owned.
  - **Expired/missing connection (a related but distinct issue):** even
    with a valid, active owner, an individual **connection** used by the
    flow can expire (password rotation, MFA re-auth requirements, or the
    connection being manually removed) independent of the owner's account
    status.
- **Correct pattern:**
  - **Always assign multiple owners** to any flow of real business
    importance — never a single point of failure. (One offboarding
    procedure explicitly recommends against dedicated service accounts
    for this specific purpose due to identity/audit-trail complexity in
    that context — weigh this against the service-principal
    recommendation in #4, which is the more platform-native
    stability mechanism for mission-critical automation specifically;
    the two aren't contradictory so much as suited to different
    situations, see #4 for when the service-principal route is preferred.)
  - **Before deactivating any user**, run a check for flows they own and
    reassign ownership as part of the offboarding process — don't wait to
    discover broken flows after the fact.
  - **Regularly audit connection references** in the Power Platform Admin
    Center rather than only discovering a stale connection when a flow
    starts failing in production.
- **Scope:** Any flow considered business-critical or with any single-
  person dependency.
- **Severity trigger:** Any flow with only one owner and no auditing
  cadence — treat this as a gap to flag proactively, not just react to
  after an owner leaves.
- **Live-search trigger:** Low — this is a stable, long-standing
  operational concern with well-documented admin tooling.

## 4. When to use a service principal instead of a human owner
- **Default habit:** Defaulting to a named individual (often whoever built
  the flow) as owner for every flow, including mission-critical,
  enterprise-wide, or pipeline-deployed flows.
- **Why it matters:** A **service principal** (a non-human security
  identity representing an application/service) can own and run flows,
  specifically insulating flow ownership from the lifecycle of any
  individual employee — the flow's stability no longer depends on one
  person's employment status, role changes, or premium license
  assignment.
- **When Microsoft's own guidance recommends this specifically:**
  - Mission-critical flows serving departmental or enterprise-wide
    scenarios.
  - Flows deployed across Dev/Test/Production via DevOps pipelines.
  - Situations where losing a flow if "the owner leaves the organization
    or their role changes" or "the owner's premium license is unassigned"
    would be a real operational risk.
- **Setup requires deliberate steps, not a toggle:** create a service
  principal application user representing the Entra ID service principal,
  **share the flow's connections with that service principal application
  user** (connections need to be explicitly shared for the SPN to run the
  flow successfully), then change the flow's owner to the service
  principal application user.
- **Caveat:** a service principal application user is non-interactive and
  unlicensed in the traditional sense — it's subject to **non-licensed
  user limits** and has distinct licensing/request-limit implications
  (cross-reference the Power Apps skill's `licensing-and-limits.md` for
  the general volatility of licensing specifics — verify current
  non-licensed-user request limits live if this is decision-critical for
  a high-volume mission-critical flow).
- **Scope:** Mission-critical or enterprise-wide flows, and any flow
  managed through a CI/CD deployment pipeline.
- **Severity trigger:** Any flow where the business impact of ownership
  disruption would be significant — evaluate service-principal ownership
  proactively for these, don't wait for an incident to trigger the
  conversation.
- **Live-search trigger:** Medium — service principal support for flows
  is a newer capability area that has continued to develop; verify
  current setup steps and licensing implications live before implementing.

## 5. Sharing/co-owner model — permissions people don't expect
- **Default habit:** Assuming any added co-owner has identical rights and
  access to everything the original owner has, including the ability to
  edit any connection's credentials.
- **Why it matters:** Owners **can use** a shared connection within the
  flow, but **cannot modify the credentials** of a connection that a
  *different* owner created — an important nuance for troubleshooting
  "why can't I fix this connection even though I'm listed as an owner"
  situations. Additionally, sharing a flow lets the sharer choose between
  the invoked user's own connections or the built-in connections of
  whoever created the flow — this choice has real behavioral consequences
  for run-as identity (cross-reference #1).
- **Run-only sharing** is a distinct, more restrictive option — it
  restricts a person to running the flow without viewing run history or
  making changes, appropriate for end-users who need to trigger a flow but
  shouldn't be able to edit it.
- **A SharePoint-specific convenience:** a SharePoint list can be added as
  a co-owner of a flow, automatically granting edit access to the flow to
  everyone with edit access to that list — useful specifically when the
  flow is SharePoint-connected; use a security group instead for every
  other case (this list-as-co-owner mechanism is not available in GCC
  High/DoD tenants — verify live if working in a government cloud tenant).
- **Removing an owner:** when removing an owner whose credentials are used
  by the flow's connections, the connections' credentials need to be
  updated/reassigned at the same time, or the flow will break the same way
  described in #3.
- **Scope:** Any flow with more than one owner, or shared with run-only users.
- **Severity trigger:** Any flow-sharing decision — confirm which sharing
  model (co-owner vs. run-only) actually matches the intended access level
  before sharing, since the two have meaningfully different capabilities.
- **Live-search trigger:** Low-medium — the GCC High/DoD exception is
  worth verifying live for government-cloud-specific work.

## 6. Environment variables deployed without their runtime values set
- **Default habit:** Deploying a solution containing environment variables
  to a new environment (test/production) and assuming the flow will work
  immediately, the same as it did in dev.
- **Why it breaks:** A solution deployment carries the environment
  variable's **definition** — its schema (name, type, default) — but not
  necessarily a populated **value** for the target environment. If the
  value is left blank in the new environment (common when only the
  definition was deployed via solution, with no separate step to populate
  the value), any flow referencing that variable fails or behaves
  incorrectly, referencing an empty/null value it was never designed to
  handle. This is the flow-specific manifestation of the same underlying
  issue the Power Apps skill's `alm-and-environments.md` #1 describes for
  hardcoded values — except here the failure mode is a **blank** value
  post-deployment rather than a value pointing at the wrong environment.
- **Correct pattern:** Always set environment variable **values** as an
  explicit post-deployment step — manually, or via a DevOps release
  pipeline task — never assume a solution import alone populates them.
  Monitor and verify key environment variable values specifically during
  UAT, before go-live, rather than discovering a blank value the first
  time the flow runs in production.
- **Scope:** Any flow using environment variables, deployed via solution
  to more than one environment.
- **Severity trigger:** Every solution deployment to a new environment —
  treat verifying environment variable values as a standard deployment
  checklist item, not an optional check.
- **Live-search trigger:** Low.

## 7. Untracked failures in loops/parallel branches — a deployment-adjacent
   reminder, not a new mechanism
- Cross-reference `error-handling-and-retry.md` and
  `concurrency-and-parallelism.md` directly — this is the same underlying
  issue (missing Configure Run After / Try-Catch structure) reframed as an
  ALM/production concern: a flow without explicit error handling in loops
  or parallel branches can "succeed" in run history while having silently
  done nothing useful, which is exactly the kind of gap that's invisible
  during dev testing (small, controlled data) and only surfaces at
  production scale/volume.

## 8. Solution ownership itself — a known rough edge
- **Default habit:** Assuming that because flow ownership can be
  reassigned relatively easily (per #3), the same is true of **solution**
  ownership.
- **Why it matters:** Changing the owner of a *solution* (as opposed to an
  individual flow within it) is reported as a **known rough edge** in the
  platform — there isn't a simple, direct mechanism for it, in contrast to
  flow ownership (which can be changed straightforwardly for solution-
  aware/"team flows") and canvas app ownership (transferable via a
  connector). This is described as an area Microsoft has incrementally
  improved but hasn't fully resolved.
- **Correct pattern:** Don't assume solution-level ownership transfer will
  be as simple as flow-level reassignment when planning a personnel
  transition — verify current tooling/support live before committing to a
  transition plan that depends on it, since this is an area under active
  platform development.
- **Scope:** Any environment with solution ownership tied to a specific
  individual who may leave or change roles.
- **Severity trigger:** Any planned offboarding or role change involving
  someone who owns solutions (not just flows) — flag this explicitly
  during transition planning rather than assuming it works the same as
  flow reassignment.
- **Live-search trigger:** Medium-high — explicitly flagged by the source
  material as an area of ongoing platform improvement; verify current
  capability before relying on this file's characterization.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Trigger type identified (manual/automated/scheduled) before
      diagnosing any "works for one user, not another" behavior
- [ ] Connection references consolidated to one-per-service unless a
      genuine multi-tenant/multi-site need justifies more
- [ ] Every business-critical flow has multiple owners, not a single
      point of failure
- [ ] Service principal ownership evaluated for mission-critical or
      pipeline-deployed flows, not defaulted to a named individual
- [ ] Offboarding process includes a flow-ownership reassignment step
      before an owner's account is deactivated
- [ ] Sharing model (co-owner vs. run-only) deliberately matched to the
      intended access level, not defaulted without consideration
- [ ] Environment variable **values** (not just definitions) verified as
      populated post-deployment, checked explicitly during UAT
- [ ] Loops/parallel branches have explicit error handling (cross-
      reference `error-handling-and-retry.md` and
      `concurrency-and-parallelism.md`) so failures can't hide behind an
      apparently "successful" run
- [ ] Solution-level ownership transition planned for explicitly and
      verified live, not assumed to work like flow-level reassignment
