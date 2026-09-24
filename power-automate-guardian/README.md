# Power Automate Guardian

**Status: not started.**

This folder is a placeholder for the Power Automate companion to
[`power-apps-guardian`](../power-apps-guardian), following the same
research-first, category-by-category process once work begins.

## Planned categories (from earlier planning, subject to change)

- Trigger/action delegation-adjacent issues — "Get items" filter query
  limits, pagination not enabled, threshold defaults silently truncating
  results
- Concurrency control — Apply to Each running sequentially by default,
  degrading performance at scale
- Connector authentication/connection reference issues in solutions vs.
  unmanaged flows
- Throttling across HTTP connectors and licensing tiers (cross-reference
  `power-apps-guardian/references/licensing-and-limits.md` — API quotas
  are shared between Power Apps and Power Automate on the same license)
- Flow timeout limits, retry policy defaults, error handling patterns
  (Scope + Configure Run After)
- Nested loops / recursive flow performance traps

## When work begins

Follow the same process as `power-apps-guardian`: research each category
live, build a schema-consistent reference file, sync it into this folder's
`SKILL.md`, and keep a verification protocol for anything volatile
(licensing, quotas, connector classifications — likely to overlap
significantly with the Power Apps skill's own licensing file).
