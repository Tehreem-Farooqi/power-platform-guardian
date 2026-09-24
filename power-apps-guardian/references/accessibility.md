# Power Apps Accessibility Reference

Accessibility is also a licensing-adjacent risk category worth naming up
front: for many organizations (government, education, larger enterprises)
accessibility compliance (WCAG-aligned) is a **legal/procurement
requirement**, not a nice-to-have — flag this explicitly for any app built
for such contexts rather than treating accessibility purely as UX polish.

## Reference table schema (applied per pattern below)
| Field | Purpose |
|---|---|
| Pattern name | Short identifier |
| Default AI / naive habit | What gets written without this knowledge |
| Why it matters | Who is excluded/impacted and how |
| Correct pattern | The fix / correct setup |
| Scope | Where this applies |
| Severity trigger | When this bites |
| Live-search trigger | Hardcode-stable, or verify live? |

---

## 1. Missing or default screen names
- **Default habit:** Leaving screens with default names (Screen1, Screen2)
  or unhelpful custom names.
- **Why it matters:** When a screen loads, screen readers announce its
  name first — it's the very first thing a screen-reader user hears, and
  it's how they orient themselves after navigation.
- **Correct pattern:** Give every screen a meaningful, descriptive name.
  Note a specific interaction trap: if `SetFocus` is called immediately on
  screen load, the screen name announcement gets interrupted/skipped —
  if a visible title and context announcement matter, use a live region
  (see #5) instead of relying solely on the screen-name announcement.
- **Scope:** Universal, every screen.
- **Severity trigger:** Every app — this is a zero-cost, always-applicable
  fix, no reason not to do it by default.
- **Live-search trigger:** Low.

## 2. Uncontrolled/illogical tab and reading order
- **Default habit:** Placing controls freely by visual design without
  considering the order screen readers and keyboard users will encounter
  them in.
- **Why it matters:** Screen reader and keyboard navigation order is
  determined by each control's **X/Y position** (top-to-bottom, then
  left-to-right for same-vertical-position controls) — **not** by the
  order controls appear in the Studio tree view, and **not** by visual
  size. A control placed oddly (even if it looks fine visually) can end up
  read in a confusing, illogical sequence.
- **Correct pattern:**
  - Structure controls hierarchically using **Container controls** to
    manage grouping and order deliberately, rather than relying on
    incidental X/Y placement.
  - Follow either a "Z" order (across, then down) or reverse-N order
    (down, then across) — pick one and apply consistently.
  - For the rare case where visual order and desired tab order must
    differ, don't fight this with unsupported hacks — **custom tab
    indexes greater than zero are a retired feature**: any TabIndex value
    greater than zero is now silently treated as zero, so this is not a
    workaround anymore. Instead, use the documented technique: wrap the
    control that should come first in the reading order in a Container,
    and position that Container's Y value above the other control — this
    genuinely reorders both tab and screen-reader sequence, not just
    visual position.
  - Set `TabIndex = 0` on genuinely interactive controls (buttons, text
    inputs, combo boxes — this is their default). Set `TabIndex = -1` on
    non-interactive Labels, Images, Icons, and Shapes **unless** they're
    being used interactively (e.g. an Image acting as a button), in which
    case set `TabIndex = 0` and give it a proper `AccessibleLabel`.
  - Enable the **"Simplified tab index"** app setting.
  - **Known testing caveat:** in Studio Preview mode, control reading
    order does **not** update live for performance reasons — the correct
    order only reflects accurately once the app is **published and run**.
    Don't judge reading-order correctness from Preview mode alone.
- **Scope:** Universal.
- **Severity trigger:** Any screen with more than a couple of controls,
  and especially any screen with dynamically-positioned controls (X/Y
  driven by a formula) — note that dynamic position changes do **not**
  update the screen-reader/tab navigation order live, which is its own
  distinct trap worth flagging for animated/dynamic layouts.
- **Live-search trigger:** Low — documented, stable platform behavior.

## 3. Missing or unhelpful `AccessibleLabel`
- **Default habit:** Leaving `AccessibleLabel` blank on Image, Icon, and
  Shape controls, or on any control lacking visible text.
- **Why it matters:** An empty `AccessibleLabel` on Image/Icon/Shape
  controls actually **hides that control from screen reader users
  entirely** — which is correct behavior for a purely decorative image,
  but a real accessibility failure for an interactive one (a clickable
  icon button with no label is invisible and unusable to a screen-reader
  user).
- **Correct pattern:** For any control conveying meaning or providing
  interactivity without visible accompanying text, set a clear, descriptive
  `AccessibleLabel`. For purely decorative elements, leaving it empty is
  the *correct* choice, not an oversight — the skill should distinguish
  these two cases rather than flagging every empty AccessibleLabel as an
  error.
- **Scope:** Universal — Image, Icon, Shape controls specifically; also
  relevant for form input labeling generally (ensure input elements are
  labeled on screen).
- **Severity trigger:** Any interactive non-text control.
- **Live-search trigger:** Low.

## 4. Color contrast
- **Default habit:** Customizing colors for branding without checking
  contrast ratios.
- **Why it matters:** Low-contrast text is difficult or impossible to read
  for users with low vision, and fails standard accessibility guidelines.
- **Correct pattern:** Ensure a contrast ratio of **4.5:1 or greater**
  between text and background for any customized color scheme. Power
  Apps' built-in themes are already designed to meet accessibility
  standards — flag custom color overrides specifically as the point where
  this needs manual verification (using a contrast-checking tool), since
  default themes don't need this check.
- **Scope:** Any app with custom (non-default-theme) color choices.
- **Severity trigger:** Any custom branding/color palette applied.
- **Live-search trigger:** Low — 4.5:1 is a stable, standard WCAG figure,
  not Power-Apps-specific.

## 5. Dynamic content changes with no announcement
- **Default habit:** Updating a Label's text dynamically (e.g. a status
  message, a validation error, a loading state) with no mechanism for
  screen readers to notice the change.
- **Why it matters:** A sighted user sees a status message appear; a
  screen-reader user navigating elsewhere on the screen has no idea
  content changed at all unless something announces it.
- **Correct pattern:** Use the Label control's **`Live`** property:
  - `Off` — no announcement (default; fine for non-critical, non-dynamic text).
  - `Polite` — the screen reader finishes its current sentence before
    announcing the change. Use for most status updates.
  - `Assertive` — interrupts current speech immediately to announce.
    Reserve for urgent/critical messages (e.g. a save failure) since
    overuse is disruptive and can itself become an accessibility problem.
  - Cross-reference: this is also the fix for the SetFocus-interrupting-
    screen-name-announcement issue in #1 — use a visible title Label set
    to a live region rather than relying purely on the screen name being
    read.
- **Scope:** Any screen with dynamic status/validation messaging.
- **Severity trigger:** Any form with client-side validation feedback, any
  screen with async loading states or save confirmations.
- **Live-search trigger:** Low.

## 6. No heading structure
- **Default habit:** Using Labels purely for visual styling (bigger font,
  bold) without setting semantic heading roles.
- **Why it matters:** Screen reader users commonly navigate by jumping
  between headings to quickly understand page structure — without
  semantic roles set, a screen reader has no way to distinguish a heading
  from a large-font decorative label.
- **Correct pattern:** Set the Label control's **`Role`** property.
  Exactly **one `Heading1`** per screen, used for the main heading.
  `Heading2` for subheadings; `Heading3`/`Heading4` for finer hierarchy as
  needed. Use `Default` for normal body text.
- **Scope:** Universal, any screen with meaningful text hierarchy.
- **Severity trigger:** Any screen with more than a trivial amount of
  text content or sectioned information.
- **Live-search trigger:** Low.

## 7. Interactive HTML content
- **Default habit:** Using HTML text controls (or embedding HTML into
  other controls) for interactive elements, not just static display.
- **Why it matters:** Power Apps **does not support accessibility for
  custom interactive HTML elements** — an interactive element built this
  way is invisible to accessibility tooling and screen readers, and the
  Accessibility Checker specifically flags this.
- **Correct pattern:** Use HTML text controls only for static, non-
  interactive display content (cross-reference `performance.md` #7's note
  on HTML text controls being useful for dense read-only layouts — that
  use case is fine; adding interactivity inside that HTML is not). Use
  standard interactive controls (Button, etc.) for anything the user needs
  to interact with.
- **Scope:** Any app using HTML text controls.
- **Severity trigger:** Any HTML control containing clickable/interactive
  elements rather than pure display content.
- **Live-search trigger:** Low.

## 8. Not running the built-in Accessibility Checker
- **Default habit:** Treating accessibility as something checked manually
  or skipped, unaware a built-in tool exists.
- **Correct pattern:** Run the **Accessibility Checker** in Power Apps
  Studio before considering an app complete. It detects screen-reader and
  keyboard issues automatically, explains why each is a problem for a
  specific disability, and suggests fixes — including catching several of
  the items above (default screen names, autostart audio/video, HTML
  accessibility issues) automatically. It does not fully replace manual
  review (e.g. it flags contrast issues but the actual ratio check needs a
  separate contrast tool), but it should be the first pass on any app.
- **Scope:** Universal — recommend running this on every app nearing
  completion, not just ones with explicit accessibility requirements.
- **Severity trigger:** Always — zero-cost to run, meaningful value.
- **Live-search trigger:** Low.

## 9. Autostart media
- **Default habit:** Leaving `Autostart` set to `true` on Audio/Video
  controls.
- **Why it matters:** Auto-playing media can distract or disorient users,
  particularly those with cognitive or attention-related disabilities, and
  can interfere with screen reader audio.
- **Correct pattern:** Set `Autostart` to `false`; let the user choose to
  play media themselves.
- **Scope:** Any app with embedded Audio/Video controls.
- **Severity trigger:** Always — flagged directly by the Accessibility
  Checker as a warning.
- **Live-search trigger:** Low.

---

## Verified screen reader / platform combinations (verify live if
## specific compatibility is decision-critical)
Reported verified combinations at time of research: JAWS with Microsoft
Edge; Narrator with Microsoft Edge; NVDA with Google Chrome/Firefox;
TalkBack with Google Chrome/Power Apps mobile; VoiceOver with Power Apps
mobile/Safari (macOS/iOS/iPadOS). Screen reader/browser compatibility
matrices shift with both screen reader vendor updates and Power Apps
platform updates — treat this list as a starting point, not a guarantee,
and verify live for any deployment where a specific combination is
contractually or legally required.

---

## Summary checklist (quick reference for the skill to apply)
- [ ] Every screen has a meaningful, non-default name
- [ ] Control layout/Containers deliberately structured for logical
      tab/reading order — not left to incidental X/Y placement
- [ ] No reliance on TabIndex > 0 (retired, no longer functional);
      Container-repositioning technique used instead if custom order needed
- [ ] AccessibleLabel set on all meaningful Image/Icon/Shape controls;
      deliberately left empty only for genuinely decorative ones
- [ ] Custom color choices checked against 4.5:1 contrast ratio
- [ ] Dynamic status/validation messages use Live (Polite/Assertive)
      appropriately, not silently updated with no announcement
- [ ] Heading roles (Heading1/2/3/4) set for meaningful text hierarchy;
      exactly one Heading1 per screen
- [ ] No interactive elements embedded inside HTML text controls
- [ ] Accessibility Checker run before considering the app complete
- [ ] Autostart disabled on all Audio/Video controls
