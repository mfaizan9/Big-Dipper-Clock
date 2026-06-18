# Accessibility Notes — Big Dipper Clock

Target: WCAG 2.1 AA (AAA where reasonable). Human screen-reader QA is still required
(VoiceOver/NVDA/JAWS) — automated review and code review do not replace it.

## Structure & landmarks
- Single `<h1>` is rendered by the `<kl-unl-masthead>` component (sim title). The sim
  does not add a competing `h1`.
- `<main class="app-layout">` wraps the content. Each panel is a `<section>` with an
  `<h2>` (`Northern Sky Diagram` — visually hidden, since the original sky panel has no
  visible title; `Time and Date Controls` — visible). Heading order is h1 → h2, no skips.
- The masthead's Reset / Help / About live in its own `<nav>` (Shadow DOM); the modal
  dialog there manages its own focus trap and Escape — the sim does not fight it. Reset
  is wired via the bubbling `sim-reset` event (no second Reset button is added).

## Text alternatives (1.1.1)
- Both canvases use `role="img"` + `aria-describedby`.
- The **sky** canvas has a polite live region (`#sky-desc`) describing the current view,
  e.g. *"Northern sky looking due north on March 19 at 1:17 PM. The Big Dipper, Little
  Dipper and Cassiopeia circle the North Star. It is currently daytime (sky bright)."*
- The **clock** canvas has a description (`#clock-desc`), e.g. *"Clock reads 06:27 AM.
  Daylight saving time is in effect."* using the original 12-hour "traditional time"
  string from `Time Of Day Panel.as`.
- Descriptions update on every committed state change, not on every drag tick.

## Color & contrast (1.4.1 / 1.4.3 / 1.4.11)
- All chrome uses the KL-UNL palette custom properties. Body text ≥ 4.5:1.
- **No state is encoded by color alone.** Day/twilight/night is always also stated in
  words in the live region ("daytime (sky bright)" / "twilight" / "night"). The DST state
  is shown as the **text** placard "daylight saving time in effect" (not a color), and is
  also spoken in the clock description. Constellations are labeled by name (when "show
  details" is on) and named in the description regardless.
- The sky/star/ground colors reproduce the original exported gradients (educational
  day/night cue); no original color was repurposed as the sole signal, so no color
  remap was required.

## Keyboard (2.1.1 / 2.1.2 / 2.4.7)
- Everything is operable by keyboard in a logical tab order; focus rings come from
  `kl-unl.css` `:focus-visible` (plus a custom ring on the day-of-year slider). No traps.
- **Time of day:** hour and minute `<input>` fields (digits only, like the AS `restrict
  = "0-9"`). Commit on Enter or blur; invalid input restores the previous display
  (matches `setTimeByTextFields`). Each field has an associated (visually hidden) label.
- **Day of year:** a day `<input>` + a real `<select>` for the month (both labeled), and
  a **fully keyboard-operable** day-of-year slider strip:
  - `role="slider"`, `tabindex="0"`, `aria-valuemin=0`, `aria-valuemax=364`,
    `aria-valuenow`, and `aria-valuetext` (e.g. "March 21 (day 80 of 365)").
  - Left/Down −1 day, Right/Up +1 day, PageDown/PageUp ∓7 days, Home → Jan 1,
    End → Dec 31. Tab moves away normally; the slider is never a focus trap and is not
    blocked by canvas pointer handlers.
- **Clock hands** are pointer-draggable (mouse + touch); the equivalent keyboard path is
  the HH:MM fields, which set the identical state.
- **set to system clock** and **show details** are a native `<button>` and
  `<input type="checkbox">`.

## Pointer / touch (works without hover)
- All dragging uses Pointer Events, so mouse and touch share one path. Draggable surfaces
  (`#clock-canvas`, `#doy-slider`) set `touch-action: none` so dragging doesn't scroll the
  page. Pointer coordinates are mapped back through the canvas's current display scale, so
  the drag/snapping math operates in the original Flash stage coordinates at any size.
- No information is hover-only. Interactive targets meet the ≥44 px minimum (`.button`
  min-height 2.75rem; checkbox row min-height; slider 2.4rem tall with a generous hit
  area).

## Timing / motion (2.2.2 / 2.3.3)
- There is **no continuous animation** in this sim (see CONVERSION_NOTES.md), so there is
  nothing that moves > 5 s and nothing that flashes. `prefers-reduced-motion` is still
  detected and honored (no motion is introduced under it).

## Text size & zoom (1.4.4 / 1.4.10)
- Body copy is ≥ 1.05rem (headings/labels scale up), sized in rem/em so it tracks the
  browser font setting. The layout uses relative units and CSS grid/flex; it reflows
  without clipping at 200% zoom and down to phone-portrait widths (single column).
- The canvases keep their original internal coordinate systems and are scaled by CSS
  (`width:100%; height:auto`), preserving aspect ratio. Canvas-painted text (clock hour
  numbers, cardinal labels, constellation labels) scales with the canvas; the spoken
  equivalents live in the HTML live regions, which zoom independently. Clock hour labels
  and constellation names that are baked into the canvas cannot expose per-glyph
  semantics, which is why the live-region descriptions exist.

## Forms & language
- `<html lang="en">`. Every input has a real `<label>` (or is inside a
  `<fieldset>`/`<legend>`: "the time of day:", "the day of year:").

## Known limitations / QA still needed
- Canvas-rendered glyphs are not individually selectable; the live regions are the
  accessible equivalent. Confirm with a real screen reader that the descriptions are
  announced at a comfortable cadence and that the slider value text reads well.
- DST detection depends on the viewer's OS/browser time zone (as in the original).
