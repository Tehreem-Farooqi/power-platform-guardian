# Verification Protocol

This file exists because a reference skill full of confident-sounding facts
is only as good as its willingness to admit when it doesn't actually know
something current. This protocol is what stands between "the skill sounds
authoritative" and "the skill is actually reliable."

## The core problem this solves
Every reference file in this skill was built from research at a specific
point in time (each file/section is marked with when it was last verified,
where volatility matters). Power Platform changes constantly — connector
classifications get reclassified, licensing plans get discontinued
(see `licensing-and-limits.md`'s Per App Plan example), delegation support
expands, offline capability limitations shrink. A skill that treats its
own cached knowledge as permanently authoritative will confidently give
stale or wrong answers exactly when it matters most — for a licensing
decision, a production deployment plan, or a specific numeric limit someone
is about to build around.

## Mandatory rules

1. **Every reference file carries a live-search-trigger rating** (Low /
   Medium / Medium-high / High) per item or per file. Before stating
   anything rated Medium-high or High as fact, search current sources
   first. Don't skip this because the cached answer "sounds right" or
   because searching feels unnecessary for a seemingly simple question —
   the rating exists specifically because these areas have moved before.

2. **Never state a specific number (price, quota, storage limit, row
   count) with confidence without either (a) having just verified it live,
   or (b) explicitly flagging it as potentially stale** — e.g., "this file
   cites ~40,000 requests/day as of last check, but Microsoft has revised
   this before; verify current limits before relying on this number for a
   production decision." A vague-but-honest answer beats a precise-but-
   possibly-wrong one.

3. **When a reference file's content conflicts with what a live search
   returns, trust the live search** — but say so explicitly rather than
   silently overriding. E.g.: "This skill's cached notes said X, but
   current documentation says Y — going with Y since it's more recent."
   This keeps the person in the loop about drift, which is also useful
   signal for updating the reference file itself later.

4. **Distinguish three confidence levels explicitly in responses**, rather
   than presenting everything in the same confident tone:
   - **Stable/settled** — mechanism-level platform behavior unlikely to
     change soon (e.g. "a SharePoint Lookup column returns a Record, not a
     scalar" — this is fundamental to how the connector works).
   - **Current-but-verify** — true as of last check, but the kind of thing
     Microsoft revises (specific quotas, plan names, prices, connector
     classifications).
   - **Uncertain/thin** — items flagged High live-search-trigger for having
     had thin documentation to begin with (e.g. Azure Table Storage
     delegation, nested-component performance cost) — say plainly "this is
     less documented; here's my best current understanding, verify before
     relying on it heavily."

5. **Don't manufacture false precision to sound authoritative.** If a
   number, limit, or behavior genuinely isn't confirmed by a reliable
   source, say "I don't have a confirmed number for this — here's what I'd
   check" rather than stating a plausible-sounding figure. This applies
   even under time pressure or when a person clearly wants a quick answer.

6. **Cross-reference before contradicting.** Several reference files
   deliberately reference each other (e.g. `offline-and-mobile-sync.md`'s
   sync-reconciliation loop uses ForAll+Patch, an intentional exception to
   `performance.md`'s general rule). Before flagging something as "wrong"
   or overriding a pattern, check whether a cross-reference note already
   explains the apparent conflict — the skill is deliberately not
   internally naive-consistent everywhere, because real engineering
   tradeoffs aren't either.

7. **When genuinely unsure whether this skill's guidance still applies**
   (e.g., discussing a connector or feature not covered in depth, or a
   scenario that doesn't clearly map to an existing reference file),
   say so directly rather than stretching an adjacent rule to fit. A
   confident wrong answer extrapolated from a similar-but-different case is
   worse than "this specific scenario isn't covered here — let me check
   current documentation."

## What this protocol is NOT for
This isn't a license to hedge everything into uselessness. Stable,
mechanism-level facts (how SharePoint Lookup columns behave, why
ForAll+Patch is slow, what `With()` scope does) don't need a search-first
disclaimer every time — that would make the skill exhausting to use. The
protocol exists specifically for the categories most likely to have moved:
prices, quotas, plan names, connector classifications, and anything this
skill's own files flag as Medium-high/High volatility.

## Practical shorthand for applying this mid-conversation
Before finalizing any answer that draws on this skill, ask:
- Does this involve a number, price, quota, or plan name? → verify live if
  rated above Low.
- Does this involve "is X still true/available/supported"? → verify live,
  regardless of rating, since availability questions are exactly the kind
  of thing that silently changes.
- Does this involve a mechanism (how something works, why a pattern
  breaks)? → cached knowledge is usually fine, but say so if genuinely
  uncertain.
- Am I about to state something specific that I can't actually trace back
  to a reference file or a fresh search? → that's the signal to stop and
  either search or say "I'm not certain about this specific detail."
