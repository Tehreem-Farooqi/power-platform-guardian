# Power Apps Guardian

A Claude Skill that catches the class of Power Apps problems you don't find
out about until you're already deep into building — delegation failures,
performance/memory issues, licensing surprises, silent error handling gaps,
column-type mismatches, variable scope bugs, offline sync issues, ALM/
environment drift, component reuse traps, and accessibility gaps.

## Why this exists

AI can write syntactically correct Power Fx all day. What it can't do
without help is know that `Filter()` against a SharePoint list with a
`IsBlank()` predicate silently stops delegating, or that a licensing plan
referenced in half the tutorials online was discontinued in January 2026,
or that editing a shared component inside a consuming app silently
disconnects it from future updates. This skill exists to close that gap —
not by guessing harder, but by checking real, researched reference content
before finalizing code, and by knowing when to stop trusting its own cached
knowledge and go verify something live.

**This is a work of two developers built from real research, not a
single AI-generated pass.** Every reference file was built from live web
research at the time of writing, cross-checked against Microsoft
documentation and community-reported behavior, and organized into a
consistent, checkable format — not written from training-data memory.

## What's covered (Power Apps canvas apps)

| File | Covers |
|---|---|
| `verification-protocol.md` | **Read first.** How to handle volatile information and avoid stating stale facts confidently. |
| `delegation.md` | Connector-by-connector delegation rules and traps (Dataverse, SQL Server, SharePoint, Salesforce, Dynamics 365, Excel/OneDrive, Azure Table Storage, legacy CDS) |
| `performance.md` | Memory/performance anti-patterns — Batch Patch vs. ForAll+Patch, N+1 queries, OnStart bloat, collection sizing, control count |
| `licensing-and-limits.md` | Standard vs. premium connector licensing, plan structure, Dataverse capacity cost, API quotas — **high volatility, verify live** |
| `error-handling.md` | IfError/OnError patterns, Patch/SubmitForm failure handling, error logging |
| `column-data-type-quirks.md` | SharePoint internal vs. display names, Choice/Lookup/Person column comparison rules |
| `variable-scope.md` | Global vs. context vs. `With()` scoping, component scope, naming conventions |
| `offline-and-mobile-sync.md` | Dataverse built-in offline-first vs. manual SaveData/LoadData, conflict resolution, change-queue patterns |
| `alm-and-environments.md` | Environment variables, connection references, managed/unmanaged solutions, deployment pitfalls |
| `component-reuse.md` | Component Libraries, the silent-local-copy trap, property-loss-on-update bugs |
| `accessibility.md` | Screen reader order, TabIndex, contrast, live regions, heading roles, Accessibility Checker |

**Not covered:** Power Automate (planned as a separate, parallel skill),
AI Builder integration, Power Pages/portals specifics, deep PCF/code
component development.

## How each file is structured

Every file (except `offline-and-mobile-sync.md`, which uses a route-based
structure instead) follows the same schema so nothing gets applied
inconsistently:

| Field | Purpose |
|---|---|
| Pattern/rule name | Short identifier |
| Default AI / naive habit | What gets written without this knowledge |
| Why it breaks | The actual mechanism |
| Correct pattern | The fix, with real code |
| Scope | Where this applies |
| Severity trigger | When this actually matters |
| Live-search trigger | Whether to trust cached content or verify live |

That last field is the important one — it's how the skill knows when its
own knowledge might be out of date.

## Installation

This skill lives inside the [`power-platform-guardian`](../) repo, alongside
its planned Power Automate companion.

1. Clone the `power-platform-guardian` repo (or download just this
   `power-apps-guardian/` folder).
2. Place this folder wherever your Claude setup looks for skills (e.g.
   `.claude/skills/power-apps-guardian/` for Claude Code, or upload per
   your platform's skill mechanism).
3. The skill activates automatically when working on Power Apps canvas
   app formulas, data source design, or app structure — no manual
   invocation needed if your setup auto-triggers on relevant `SKILL.md`
   descriptions.

## How to use it in a session

Just work on your Power Apps project normally. The skill checks formulas
and design decisions against the relevant reference file(s) before
finalizing, and — critically — verifies anything flagged high-volatility
(pricing, quotas, connector classifications, offline capability limits)
against current sources rather than stating cached numbers as fact.

At the end of a session, ask for the session changelog (issue → root
cause → fix → reusable snippet) so the next person working on the app has
a record of what was caught and why.

## Status

**Power Apps side: complete**, as of the file-level "last verified" dates
noted inside each reference file (see `licensing-and-limits.md` and
`offline-and-mobile-sync.md` specifically — both are explicitly
high-volatility). This isn't a "finished forever" claim — Power Platform
changes, and parts of this skill are built to know that about themselves.

**Power Automate: not started.** Planned as a separate companion skill
covering flow-specific pain points (Apply to Each concurrency, trigger
thresholds/pagination, connection references in flows, throttling, retry
policies) — distinct enough from canvas app development to warrant its
own file set rather than being bolted onto this one.

## Contributing / maintaining this skill

This skill is meant to be a living document, not a one-time snapshot:

- If you hit a real issue this skill didn't catch, add it as a new
  pattern entry in the relevant file (or a new file, if it's a genuinely
  new category) using the same schema — don't just note it in passing.
- If you discover something in this skill is now stale (a limit changed,
  a feature graduated from preview, a plan was renamed again), update the
  file and its "last verified" note rather than leaving the old
  information in place.
- Keep the verification-protocol.md's live-search-trigger discipline
  intact when adding new content — mark new items honestly for how
  likely they are to change, don't default everything to "Low" just
  because it feels stable right now.

## License

Add your preferred license here before publishing.
