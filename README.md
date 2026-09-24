# Power Platform Guardian

Claude Skills that catch Power Platform problems before they hit
production — delegation failures, performance traps, licensing surprises,
silent error-handling gaps, offline sync bugs, and more.

Built from real, live research at the time of writing — not from AI
training-data memory — and organized so each skill knows when its own
cached knowledge might be stale and needs a live check before it's trusted.

## Skills in this repo

| Skill | Status | What it covers |
|---|---|---|
| [`power-apps-guardian/`](./power-apps-guardian) | Complete | Canvas app Power Fx — delegation, performance, licensing, error handling, column/data-type quirks, variable scope, offline/mobile sync, ALM/environments, component reuse, accessibility |
| [`power-automate-guardian/`](./power-automate-guardian) | Not started | Flow-specific pain points — Apply to Each concurrency, trigger thresholds/pagination, connection references in flows, throttling, retry policies |

Each skill is self-contained — its own `SKILL.md`, its own `references/`
folder, its own README. They share the same design philosophy (see below)
but don't depend on each other.

## Design philosophy

1. **Research-first, not memory-first.** Every reference file was built by
   searching current documentation and community-reported behavior, not
   generated from an LLM's training data. Where something is genuinely
   settled platform mechanics, it's cached for speed. Where it's the kind
   of thing Microsoft changes (pricing, quotas, plan names, connector
   classifications), it's flagged for live verification instead.
2. **Consistent, checkable structure.** Reference files use a shared
   schema (pattern → why it breaks → fix → scope → severity →
   live-search-trigger) so nothing gets applied inconsistently, and so a
   human reviewing the file can sanity-check it without reading prose.
3. **Honest about incompleteness.** A skill's `SKILL.md` says plainly what
   it does and doesn't cover. No file claims permanent, universal
   authority — see each skill's verification protocol.
4. **Living documents, not snapshots.** These are meant to be corrected
   and extended as real projects surface new issues — see each skill's
   README for contribution notes.

## Installation

Clone this repo and place whichever skill folder(s) you need wherever your
Claude setup looks for skills:

```bash
git clone <this-repo-url>
# then, per your platform:
cp -r power-platform-guardian/power-apps-guardian /path/to/your/skills/
```

See each skill's own README for details specific to that skill.

## Status

Power Apps: complete. Power Automate: planned, not yet started — will
follow the same category-by-category, research-first process once begun.
