# Conversion Notes — Big Dipper Clock (Flash AS1 → HTML5 / KL-UNL)

## Behavior model (one paragraph)

The Big Dipper Clock is a **static** (no animation loop) educational tool. The user
picks a **day of year** (1–365 via a day field + month drop-down, a draggable month
timeline strip, or by dragging the clock past midnight) and a **time of day** (HH:MM
fields or by dragging the clock hands). From those two values the sim computes a
"solar days since zero" quantity and (1) rotates the northern-sky star field (Big
Dipper, Little Dipper, Cassiopeia) rigidly around the North Star, keeping the text
labels upright, and (2) fades a daytime blue-sky layer in/out using a twilight model
for ~41° N latitude so the sky shows day, twilight, or night. A 24-hour clock face
shows the time (hour hand = one revolution per **day**; minute hand = standard 60-min
sweep). A "daylight saving time in effect" placard appears when the chosen date falls
in DST for the viewer's locale, and shifts the sky by the DST delta. "show details"
toggles the constellation labels and the orange Big-Dipper→North-Star pointer line.
"set to system clock" jumps to the current local date/time. Reset returns to day 78
(March 20), time 0.05 (01:12), labels on.

## Source of truth

Behavior is ported **verbatim** from the decompiled ActionScript:

| HTML5 (`simulation.js`)                | ActionScript source |
|----------------------------------------|---------------------|
| `setTimeOfDay/getTimeOfDay`, `setDayOfYear`, `setFracDayOfYear`, `update`, `reset`, `setToNow`, DST setup, `monthPoints` | `scripts/frame_1/DoAction.as` |
| `starGroupRotationDeg`, `skyOpacity` (twilight model + constants) | `scripts/Sky Diagram.as` |
| `drawClock`, hand geometry, `clockHour/clockMinute`, hand drag + day/hour rollover | `scripts/Time Of Day Clock.as` |
| `syncTimePanel`, `commitTimeFields`, traditional-time string, HH:MM formatting | `scripts/Time Of Day Panel.as` |
| `syncDatePanel`, `commitDateFields`, month combo, validation | `scripts/Day Of Year Panel.as` |
| day-of-year strip cursor (`setCursorDay` = `scaleFactor*(day+0.5)`), increment/drag | `scripts/Day Of Year Slider.as` |
| masthead title / Help / About | `foundation/contents.json` entry `dipperclock` |

All physical constants are copied unchanged: `monthPoints`, twilight angle
`0.12217304763960307`, latitude `0.7155849933176751`, obliquity factor
`0.39714789063478056`, the rotation expression
`(102.5 - 360*(solDaysSinceZero*366/365 - 0.49583333333333335)) % 360`, and the solar
offset `dayOfYear + timeOfDay - 78.5 + delta`.

### Visual parity check
Set to the original screenshot's state (March 19, 13:17) the rebuilt sky reproduces
the original's orientation: Cassiopeia top, Little Dipper center-left, North Star
center, Big Dipper bottom, orange pointer line up-and-to-the-right, daytime blue sky —
confirming the verbatim rotation/twilight math.

## "Static" simulation — pauseAnimation/resumeAnimation are no-ops

The AS calls `masterMC.pauseAnimation()` / `resumeAnimation()` from the drag handlers,
but **those functions are never defined anywhere in the decompiled source**, so in the
original they silently do nothing. There is no `onEnterFrame` animation of the clock or
sky. The port is therefore event-driven: a single `render()` redraws everything from
one state object after each user action. (A `Pause` control is therefore not applicable;
`prefers-reduced-motion` is still honored — there is simply no continuous motion to
reduce.)

## Asset reuse

The constellation geometry is taken **verbatim** from the exported vector shapes, which
all share one coordinate frame whose origin is the North Star / rotation pole:

* `assets/262.svg` — Big Dipper (7 stars) + Cassiopeia (5 stars) connector lines
* `assets/273.svg` — Little Dipper (7 stars) connector line (last vertex ≈ Polaris)
* `assets/271.svg` — orange dotted Big-Dipper→North-Star pointer line
* `assets/274.svg` — the white 8 px star dot

The exact path vertex coordinates from these files are embedded as data in
`simulation.js` (`BIG_DIPPER`, `CASSIOPEIA`, `LITTLE_DIPPER`, `POINTER_LINE`) and drawn
with the canvas 2D API, because the rotating star group has to composite and rotate as
one unit around the pole (so it cannot be a set of independently-cropped `<img>`s). The
sky/ground gradients reproduce the exported gradient stops from `shapes/259.svg`
(night), `shapes/260.svg` (day sky) and `shapes/277.svg` (ground). The 24-hour clock
face, ticks, and hands are code-drawn (the AS builds them on the timeline; hand styling
follows `shapes/239–243.svg`: black short fat hour hand, grey long thin minute hand).

The text-label **anchor points** in the star frame were measured from the
`DefineSprite_279_Sky Diagram` render (the labels are visual only and drive no readout).

## Deviations (Goal A vs Goal B/asset reality)

1. **Pre-existing broken `contents.json` (required fix).** The shared
   `foundation/contents.json` shipped with **invalid JSON** that prevented the masthead
   from parsing on *every* sim: raw (unescaped) newline characters inside several string
   values (e.g. the `ce_hc` and `eclipsingbinarysim` help text) and **unescaped double
   quotes** inside HTML `href="…"` attributes (e.g. the `renaissancePtolemaic` entry).
   I corrected only these syntax errors in the copied `html5/foundation/contents.json`
   (collapsed the stray control characters to spaces; escaped the attribute quotes) so
   the file parses. **No text content was changed and the `dipperclock` entry was not
   touched.** The upstream/shared `contents.json` should receive the same fix.
2. **`dipperclock` entry already present.** The provided `contents.json` already
   contained a complete `dipperclock` entry (title, version, Help, About), so no entry
   was added — matching the original sim, whose Title Bar had empty `aboutLinkageName`
   and `helpLinkageName` (it showed only "reset"). The foundation's modernized
   Help/About text is used as-is.
3. **DST detection.** Two small corrections, both of which leave behavior **identical**
   for DST-observing locales (such as the University of Nebraska's Central Time):
   - `frame_1/DoAction.as` uses `now.getYear()` (year − 1900) fed to
     `new Date(currentYear, …)`, where small year values are interpreted inconsistently.
     The port uses the true current year (`new Date().getFullYear()`) so DST detection is
     correct.
   - The AS sets the placard with `d.getTimezoneOffset() == daylightSavingOffset`. In a
     locale that does **not** observe DST, `standardOffset == daylightSavingOffset`, so
     that test is always true and the original would show "daylight saving time in effect"
     **year-round** (a factual error; the sky delta is 0 regardless). The port adds the
     guard `daylightSavingOffset !== standardOffset`, suppressing the false placard in
     non-DST locales. For DST locales the two offsets differ, so this changes nothing.
4. **No MathJax / no mathematics.** This sim contains **no equations or mathematical
   notation** anywhere in its UI (only plain numeric data values: hours, minutes, day
   number, month names, and clock hour labels). There is therefore nothing for MathJax to
   typeset, and the foundation does not ship a local MathJax library (and a CDN is
   disallowed). `foundation/kl-unl.js` is still loaded for foundation parity and
   `klunlInitEqn()` is redefined as a no-op. See ACCESSIBILITY.md.
5. **Day-of-year strip as an accessible slider.** The original timeline strip was a
   mouse-only draggable cursor (with click-to-increment auto-repeat). The port keeps the
   pointer drag (mapped through the element size) and **adds** full keyboard operation
   (`role="slider"`, arrows/Page/Home/End) plus `aria-valuetext`, as a separate component
   in `styles/styles.css` + `simulation.js`. Behavior of `setDayOfYear` is unchanged.
6. **Clock-internal text** (hour numbers 0–23, "12 am/12 pm/6 am/6 pm") is painted on the
   canvas as in the original. A live text description of the current time/date/sky is
   provided in an `aria-live` region so the canvas content is available to screen readers
   (see ACCESSIBILITY.md). These are plain labels, not mathematical notation.

## Foundation files

`kl-unl-masthead.js`, `kl-unl.css`, `kl-unl.js` are copied **byte-for-byte unchanged**.
The only content change to any foundation file is the JSON-syntax repair described in
deviation #1 (which does not alter any displayed text and does not touch this sim's
entry).
