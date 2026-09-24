# Power Apps ALM, Solutions & Environment Reference

This file covers deployment-lifecycle issues — the class of problem that
doesn't show up while building in one environment, only when moving an app
from dev to test to production. AI-written Power Fx is entirely blind to
this category unless it's told to check for it, because none of it shows
up as a formula error in Studio.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written/recommended without this knowledge |
| Why it breaks | The actual deployment-time failure mode |
| Correct pattern | The fix / correct setup |
| Scope | Where this applies |
| Severity trigger | When this actually bites |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. Hardcoded environment-specific values
- **Default habit:** Hardcoding a SharePoint site URL, SQL connection
  string, API endpoint, or email address directly into formulas.
- **Why it breaks:** The moment the app is exported from dev and imported
  into test/production, every hardcoded reference still points at the dev
  resource. The app "works" during dev testing and silently writes to (or
  reads from) the wrong environment after deployment — a genuinely
  dangerous failure mode, not just an inconvenience.
- **Correct pattern:** Use **Environment Variables** for any value that
  differs between dev/test/prod — API endpoints, email addresses, feature
  flags, SharePoint site URLs. Use **Connection References** for the actual
  data source connections (SharePoint, SQL, Dataverse, etc.) rather than
  raw connections, since connection references can be repointed at import
  time without editing the app itself.
  ```
  // Instead of hardcoding:
  Notify("Sent to admin@devcompany.com")

  // Reference an environment variable:
  Notify("Sent to " & AdminEmail.Value)
  ```
- **Scope:** Universal — applies to every app that will ever move between
  environments, which in practice means every production app.
- **Severity trigger:** Any app with more than one target environment
  (i.e., essentially all real projects — few apps stay in dev forever).
- **Live-search trigger:** Low — mechanism is stable.

## 2. Editing directly in managed (test/production) environments
- **Default habit:** Making a "quick fix" directly in the test or
  production environment because it's faster than round-tripping through
  dev.
- **Why it breaks:** Editing a component inside a managed solution creates
  an **unmanaged layer** on top of it — a hidden, environment-specific
  customization that the next managed-solution update from dev won't
  necessarily overwrite cleanly, and which isn't tracked in source control.
  This is how environments quietly drift out of sync with dev over time,
  and how "it works in prod but the code in the repo doesn't do that"
  situations happen.
- **Correct pattern:** Always develop in an **unmanaged** solution in a
  dedicated dev environment; export as **managed** for test/production.
  Never edit components directly in test/production. Setting the "Allow
  Customizations" property off on production artifacts is a recommended
  practice specifically to prevent this — though be aware it can
  complicate connection reference updates during pipeline deployment (see
  #3), so this needs coordinated setup, not a blanket toggle.
- **Scope:** Universal ALM discipline.
- **Severity trigger:** Any team with more than one environment or more
  than one developer — the risk compounds with team size and environment
  count.
- **Live-search trigger:** Low.

## 3. Connection references — the exception worth knowing
- **Default habit:** Treating all managed-solution components as equally
  "locked," including connection references.
- **Why it matters:** Connection references are a deliberate exception —
  they **can be updated without creating an unmanaged layer**, unlike
  editing a flow or canvas app directly. This is specifically what allows
  a managed solution's connections to be repointed per-environment without
  triggering the drift problem in #2.
- **Correct pattern:** Keep "Allow Customizations" enabled specifically for
  connection references (even while disabling it elsewhere) so environment-
  specific connection updates can happen cleanly during deployment/pipeline
  imports.
- **Best-practice structuring:** one connection reference per service
  (connector) per environment — don't create a separate connection
  reference for the same service in every solution. A common pattern:
  maintain a dedicated "Core Connections" solution per environment that
  other solutions declare as a dependency, so a SharePoint connection (for
  example) is defined once per environment and reused everywhere, not
  duplicated per app/flow.
- **Multi-developer nuance:** for a single developer or sequential
  multi-developer work, centralize connection references. For multiple
  developers working simultaneously in dev, solution-specific references
  may be more practical short-term, consolidated before promoting to test/
  prod. Production should always be centralized with service accounts, not
  personal connections.
- **Scope:** Universal.
- **Severity trigger:** Any solution with more than a handful of
  connections, or any team larger than one developer.
- **Live-search trigger:** Low-medium.

## 4. Solution structuring — single vs. multi-solution
- **Default habit:** No deliberate structure — components added to
  whatever solution is open, or one giant solution with no logical grouping.
- **Correct pattern:** For most standard applications, a **single-solution
  strategy** — apps, flows, environment variables, and connection
  references together in one solution — is the recommended, simpler
  approach. Reserve a **multi-solution strategy** (splitting core
  environment variables/connection references into their own solution from
  the app components) for genuinely complex, multi-app architectures — and
  be aware this creates an explicit dependency: the environment/connections
  solution must deploy before the app solution that depends on it.
- **Scope:** Universal, scales in importance with project complexity.
- **Severity trigger:** Complexity-dependent — flag the multi-solution
  approach as unnecessary overhead for a single simple app, but recommend
  it once multiple apps/flows are sharing components.
- **Live-search trigger:** Low.

## 5. Reference/seed data doesn't travel with a managed solution
- **Default habit:** Assuming that Dataverse table data the app depends on
  (lookup values, category definitions, configuration rows) will "just be
  there" after deploying the solution to a new environment.
- **Why it breaks:** Solutions carry schema, not data. A managed solution
  import will create the *table structure* in production, but any rows the
  app expects to find (e.g., a "Status" choice-backing table, or seeded
  configuration records) won't exist unless separately handled.
- **Correct pattern:** Depending on complexity: use the **Configuration
  Migration Tool** to export reference data as a schema+data package and
  import it as a deployment step; use **environment variables** for simple
  standalone config values (URLs, feature flags); or use **Power Platform
  CLI/pipeline scripts** to upsert required seed records as part of the
  deployment pipeline.
- **Scope:** Dataverse-based apps specifically.
- **Severity trigger:** Any app whose formulas assume specific reference/
  lookup data exists — check this explicitly before a first production
  deployment, since it's easy to miss until users hit a blank dropdown or
  a broken lookup in prod.
- **Live-search trigger:** Low-medium.

## 6. Secrets in environment variables — use the secret type
- **Default habit:** Storing an API key or connection string in a plain
  "text" type environment variable.
- **Why it matters:** Plain-text environment variables are visible in the
  solution and to anyone with maker access. Secrets deserve different
  handling.
- **Correct pattern:** Use the **secret environment variable type** for API
  keys and connection strings specifically. When deploying via a pipeline,
  set the value through a secure pipeline variable rather than storing it
  in plain text in a deployment settings file.
- **Scope:** Any app/flow using external API keys or credentials via
  environment variables.
- **Severity trigger:** Any use of an API key, credential, or connection
  string as an environment variable — treat as a security requirement, not
  optional hardening.
- **Live-search trigger:** Low-medium.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] No hardcoded environment-specific URLs, endpoints, or addresses —
      environment variables used instead
- [ ] Development happens in an unmanaged solution in a dev environment;
      test/production receive managed solution exports only
- [ ] No direct edits planned/made in test or production environments
- [ ] Connection references structured per the one-per-service-per-
      environment pattern, not duplicated per app
- [ ] Solution structure (single vs. multi) deliberately chosen based on
      actual project complexity, not defaulted without thought
- [ ] Reference/seed data deployment plan exists if the app depends on
      specific Dataverse rows being present
- [ ] Secrets stored using the secret environment variable type, not plain
      text
