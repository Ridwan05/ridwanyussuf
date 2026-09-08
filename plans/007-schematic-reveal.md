# 007 — Reveal the FIG. 1 schematic on first view

- **Status**: TODO
- **Commit**: aa7e4d4
- **Severity**: LOW
- **Category**: Missed opportunities (additive)
- **Estimated scope**: 1 file, ~45 lines including SVG `<g>` wrappers

## Why this is separate from 001–006

This is the only plan in the set that requires **markup changes** and carries
real browser-behaviour risk. It was deliberately left unimplemented while the six
corrective plans landed. It is genuinely worth doing — it is just not worth
doing carelessly, and it needs a browser in front of a human.

## Problem

`index.html:391-437` is a hand-built SVG diagram — FIG. 1, the delivery
lifecycle: five labelled stages (DISCOVER → STRATEGISE → BUILD → ADOPT →
MEASURE) joined by a dashed pipeline, with a dashed feedback loop returning from
MEASURE to DISCOVER under an "ITERATE ON IMPACT" caption.

It is the signature element of the hero and it is completely static. The diagram
*describes a process* — a sequence with a loop — and renders it as a still image.

## Gate result

- **Frequency**: **rare / first-time**. Seen once per visit, in the hero. This is
  the tier where the delight budget lives.
- **Purpose**: **Explanation** — the one purpose explicitly reserved for
  marketing and onboarding surfaces. Motion here demonstrates the lifecycle the
  surrounding copy claims.
- **Speed**: marketing/explanatory motion is exempt from the sub-300ms UI budget
  and may run longer.
- **Function**: an illustrative diagram, not data a user reads to act on.

Passes all four gates — the only opportunity on the page that clears the bar for
a longer, more expressive animation.

## Target

A once-only staggered reveal of the five stages left to right, then the feedback
loop. Fire it from `IntersectionObserver` with `{ once: true }` semantics.

### Markup

Wrap each stage's existing elements in a `<g class="stage">`, and the feedback
loop's three elements in a `<g class="loop">`. Add `class="schematic-anim"` to
the `<svg>`. **Move no coordinates and change no visual attribute** — wrapping
only.

```html
<!-- target — stage 01, index.html:396-401 wrapped -->
<g class="stage">
  <rect x="14" y="42" width="92" height="80" fill="none" stroke="rgba(198,161,91,0.30)" stroke-width="0.7"></rect>
  <text x="60" y="34" …>01</text>
  <circle cx="60" cy="82" r="5" fill="#D07986"></circle>
  <text x="60" y="66" …>DISCOVER</text>
  <text x="60" y="108" …>ASSESS · NEEDS</text>
</g>
```

The `<line>` at `index.html:394` stays outside the groups — it is the pipeline
the stages sit on.

### CSS

```css
/* target */
.schematic-anim .stage,
.schematic-anim .loop{
  opacity:0;
  transition:opacity 500ms var(--ease-out);
}
.schematic-anim[data-revealed] .stage,
.schematic-anim[data-revealed] .loop{opacity:1;}
.schematic-anim[data-revealed] .stage:nth-of-type(2){transition-delay:80ms;}
.schematic-anim[data-revealed] .stage:nth-of-type(3){transition-delay:160ms;}
.schematic-anim[data-revealed] .stage:nth-of-type(4){transition-delay:240ms;}
.schematic-anim[data-revealed] .stage:nth-of-type(5){transition-delay:320ms;}
.schematic-anim[data-revealed] .loop{transition-delay:420ms;}
```

80ms per stage — the upper end of the 30–80ms band, chosen because this is
explanatory rather than decorative motion and the eye should follow the sequence.
Total 420ms + 500ms = 920ms, which is acceptable on a marketing surface.

### JS

Reuse the observer pattern already in the file rather than adding a second one:

```js
/* target */
const schematic = document.querySelector('.schematic-anim');
if(schematic){
  const revealIo = new IntersectionObserver((entries, obs) => {
    entries.forEach(e => {
      if(e.isIntersecting){
        e.target.setAttribute('data-revealed', '');
        obs.unobserve(e.target); // fire once — re-animating on every scroll-by
      }                          // is an interface fighting its reader
    });
  }, {threshold: 0.25});
  revealIo.observe(schematic);
}
```

### Reduced motion

Add to the plan-003 block:

```css
.schematic-anim .stage,
.schematic-anim .loop{opacity:1;transition:none;}
```

Reduced motion here means the diagram is simply present — there is no gentler
version of a sequential reveal worth keeping.

## Do NOT animate `stroke-dashoffset` on the pipeline

The obvious instinct is to "draw" the dashed pipeline line
(`index.html:394`) and the feedback loop path (`index.html:432`) with
`stroke-dasharray` / `stroke-dashoffset`. **Both already carry
`stroke-dasharray="4 4"` for their dashed blueprint look.** Overriding
`stroke-dasharray` to draw them replaces the dashes with a solid line; animating
`stroke-dashoffset` against the existing `4 4` makes the dashes *crawl* along the
path instead of drawing it. Neither is the intended effect. Opacity on grouped
stages is the correct tool here.

Also do **not** reach for `clip-path: inset()` on the SVG `<g>` elements for a
left-to-right wipe. Percentage `inset()` on SVG child elements resolves
inconsistently across engines and is not worth the risk for this payoff.

## Repo conventions to follow

- The existing `IntersectionObserver` at `index.html:815-823` is the exemplar for
  observer style, including the explanatory comment above it
  (`index.html:812-814`). Follow its formatting and place the new observer after it.
- SVG stage blocks are already separated by `<!-- stage 0N — NAME -->` comments
  (`index.html:396`, `403`, `410`, `417`, `424`). Keep those comments outside the
  new `<g>` wrappers so the structure stays readable.
- Curve token from plan 001: `var(--ease-out)`.

## Steps

1. Apply plans 001 and 003 first.
2. Add `class="schematic-anim"` to the `<svg>` at `index.html:392`.
3. Wrap each of the five stage blocks (`index.html:396-401`, `403-408`,
   `410-415`, `417-422`, `424-429`) in `<g class="stage">…</g>`.
4. Wrap the feedback-loop `<path>`, `<polygon>` and `<text>`
   (`index.html:432-434`) in `<g class="loop">…</g>`.
5. Add the CSS from Target next to the existing `.schematic` rules
   (`index.html:202-207`).
6. Add the observer JS from Target after the existing observer at
   `index.html:823`.
7. Add the reduced-motion override into the plan-003 block.

## Boundaries

- Do NOT change any `x`, `y`, `cx`, `cy`, `r`, `width`, `height`, `d`, `points`,
  `fill`, `stroke`, `stroke-width`, `stroke-dasharray`, `font-size` or
  `letter-spacing` attribute. Wrapping only.
- Do NOT touch the `role="img"` or `aria-label` on the `<svg>`
  (`index.html:392`) — the diagram's accessible name must stay intact, and a
  screen reader must not be exposed to the reveal at all.
- Do NOT change `.schematic-caption` (`index.html:436`).
- Do NOT animate `stroke-dashoffset` or `clip-path` — see the section above.
- Do NOT add a motion library. CSS transitions plus one observer is the whole job.
- Do NOT extend this reveal to any other section of the page.
- If the SVG structure has drifted from the line references above, STOP and report.

## Verification

- **Mechanical**: `grep -c 'class="stage"' index.html` returns 5.
  `grep -n "stroke-dashoffset" index.html` returns nothing.
  `grep -n 'stroke-dasharray="4 4"' index.html` still returns 2 — the dashes survive.
- **Feel check**: this plan's payoff cannot be judged from code. Open
  `index.html` and:
  - Hard-reload with the hero in view. The five stages must appear left to right
    in an even cascade, then the feedback loop last. The rhythm should read as
    "the process runs, then it loops".
  - Scroll down past the hero and back up. It must **not** replay.
  - In DevTools → Animations at 10% playback, confirm the gaps between stages are
    even and the loop is clearly last.
  - Enable *Emulate CSS prefers-reduced-motion: reduce* and hard-reload. The
    whole diagram must be present immediately, at full opacity, with no cascade.
  - Throttle the network to Slow 3G and reload. The stages must not appear
    half-revealed and stall — if the fonts are still loading, confirm the labels
    do not reflow mid-cascade.
  - Look at it again the next day with fresh eyes before calling it done. An
    80ms-per-stage cascade is a judgement call, and 60ms or 100ms may read
    better on this diagram than the value specified here.
- **Done when**: the diagram reveals once per page load in sequence, never
  replays, is fully present under reduced motion, and the dashed blueprint
  styling of both the pipeline and the loop is visually unchanged.
