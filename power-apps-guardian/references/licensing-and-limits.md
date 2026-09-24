# Power Apps Licensing, Connector Limits & Quota Reference

> ⚠️ **This entire file is HIGH live-search volatility.** Unlike delegation
> and performance (which track platform mechanics that change slowly),
> licensing structures, plan names, and pricing track Microsoft's business
> decisions and can change on a timeline unrelated to any code or platform
> update. **Re-verify current licensing state via web search before giving
> confident licensing advice, even if this file was updated recently.**
> Last verified: 2026-09-22. Confirmed as of that date: the Per App Plan
> (~$5/user/app/month) was discontinued in January 2026.

## Reference table schema (applied per item below)
| Field | Purpose |
|---|---|
| Rule/trap name | Short identifier |
| Current state | What's true as of last verification (date-stamped) |
| Who's affected | Whose licensing/cost this touches |
| The trap | How this bites someone who doesn't know about it |
| Mitigation | What to check or do about it |
| Live-search trigger | Always High for this file — noted per-item for emphasis where it matters most |

---

## 1. Standard vs. premium connector classification
- **Current state:** Standard connectors (SharePoint, Excel Online, Outlook
  365, Microsoft Teams, OneDrive, Planner, To Do, Approvals, Office 365
  Users, Notifications) are covered by seeded Microsoft 365 entitlements —
  free to build against with no extra license. Premium connectors (SQL
  Server, Salesforce, SAP, Dataverse, Azure Blob Storage, custom connectors,
  on-premises gateway, raw HTTP to external services, and most third-party
  connectors) require a paid Power Apps license.
- **Who's affected:** **Every single user who opens the app** — not just the
  developer. This is the part that surprises people.
- **The trap:** A dev builds an otherwise-simple SharePoint app, adds one
  SQL Server lookup or one Dataverse table for convenience, and the entire
  user base now needs paid licensing to open the app at all.
- **Mitigation:** Before finalizing a data source choice, flag explicitly
  whether it's standard or premium, and confirm the dev/team understands the
  licensing consequence for every intended user — not just whether the
  connector is technically the right engineering choice.
- **Live-search trigger:** High — Microsoft can **reclassify a connector
  from standard to premium**, and this change can take effect without a
  corresponding code or platform change ("can land overnight" per community
  reporting). An app that was free to run can suddenly require licensing
  with zero changes on the developer's end. Don't assume a connector's
  classification from memory or from this doc without checking current status.

## 2. Current plan structure (verify before quoting)
- **Current state (as of 2026-09-22):** Per App Plan is discontinued
  (January 2026). Live options: **Power Apps Premium (per user)**, roughly
  $20/user/month, unlimited apps and all premium connectors; and
  **Pay-As-You-Go**, metered through Azure, no upfront seat commitment —
  useful for apps with occasional/unpredictable usage.
- **The trap:** A large amount of existing tutorials, community answers,
  and likely AI training data still reference the old Per App Plan
  ($5/user/app). Advice built on that is now stale and will misquote cost
  structure to a client or team.
- **Mitigation:** Always verify current plan names/pricing live before
  giving licensing cost guidance — don't rely on cached knowledge (including
  this file) for numbers.
- **Live-search trigger:** High, always.

## 3. Dataverse capacity as a hidden cost layer
- **Current state:** Per-user plans include an amount of Dataverse storage
  capacity (commonly cited around 250 MB/user); crossing that threshold
  requires purchasing additional capacity, priced at a premium compared to
  Azure SQL or Cosmos DB.
- **The trap:** Our own delegation guidance (see `delegation.md`) correctly
  recommends Dataverse as the best-delegating, most production-suitable data
  source. That's sound *technical* advice that can carry a real, separate
  *cost* consequence the dev didn't ask about.
- **Mitigation:** When recommending Dataverse for technical/delegation
  reasons, pair it with a one-line cost caveat — don't let a purely
  technical recommendation silently carry an unstated cost implication.
- **Live-search trigger:** Medium — capacity allowances shift with plan
  changes; verify when giving specific numbers.

## 4. API / Power Platform request quotas
- **Current state:** Requests are capped per license type per 24-hour
  period (commonly cited figures: ~40,000/user/day on premium per-user
  plans; ~6,000/user/day on lighter M365-included tiers; higher pooled
  tenant-level allocations exist for Dynamics 365-linked tenants). These
  quotas are **shared across Power Apps and Power Automate usage on that
  license** — not siloed per product.
- **The trap:** A Power Apps-only developer can burn through shared quota
  via unrelated Power Automate flows on the same license (or vice versa),
  and conversely, an app that polls or syncs data too frequently can trigger
  tenant-wide throttling that affects flows too.
- **Mitigation:** When designing polling/sync-heavy app logic, factor in
  that the request budget isn't exclusive to the app being built. Flag
  frequent OnStart/OnVisible or timer-triggered data refresh patterns as a
  quota risk, not just a performance one.
- **Live-search trigger:** Medium-high — these numbers have been revised by
  Microsoft before (they were "substantially increased in late 2021" per
  official docs) and transition-period limits vs. official limits can differ
  during enforcement rollouts. Always check current enforcement status
  rather than quoting a fixed number as permanent.

## 5. Connection count practical limit (cross-reference)
- See `performance.md` #11 for the ~30-connection-per-app guidance and the
  unresolved community debate over whether it counts distinct services or
  distinct connector instances. Relevant here too because more connections
  can mean more premium-connector exposure, not just more sign-in friction.

---

## How this file should be used
Unlike `delegation.md` and `performance.md`, this file is not meant to be
treated as reliably cached knowledge. Its job is to flag **which categories
of licensing trap exist** so the skill knows what to check — the actual
current numbers, plan names, and connector classifications should be
verified live whenever licensing advice materially affects a
recommendation (e.g., before telling someone "just use Dataverse" or before
estimating rollout cost for a user base).
