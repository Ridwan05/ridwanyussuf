# 007 — Reveal the FIG. 1 schematic on first view

- **Status**: TODO
- **Commit**: b90a6de (revised; originally written against aa7e4d4)
- **Severity**: LOW
- **Category**: Missed opportunities (additive)
- **Estimated scope**: 1 file, ~50 lines including SVG `<g>` wrappers

## Revision note — read this before implementing

The first draft of this plan hid the diagram with a plain
`.schematic-anim .stage{opacity:0}` in CSS. That is wrong, and would have shipped
a real defect: the diagram is only ever made visible by JS, so with scripting
unavailable **five of the six visual elements in the hero diagram vanish
permanently**, leaving a bare dashed line and a caption. The page currently
degrades cleanly without JS — only the nav panel and the title-block label depend
on it, and both fail safe. This would have been the first piece of actual content
gated behind script execution.

A second proposal — arm the hidden state from JS with a `data-armed` attribute —
was also rejected. It fixes the no-JS case but introduces a worse one: the SVG
renders visible in the HTML, so arming it after first paint produces a **flash of
the complete diagram, then a blank, then the cascade**. Trading a silent failure
for a visible flicker is not an improvement, and it makes the transition's
starting value depend on JS write ordering — if the arm and the reveal land in
the same task, the browser coalesces them and no transition plays at all.

The mechanism below uses `@media (scripting: enabled)` instead. It applies at
first paint (no flash), needs no JS to arm (no blank diagram), and in a browser
without support for the feature the query simply does not match — so the diagram
stays visible and the reveal is skipped. **Every failure mode now fails toward
"the diagram is visible"**, which is the correct direction.

## Why this is separate from 001–006

This is the only plan in the set that requires **markup changes**, and after the
revision above it also depends on a modern CSS media feature and on script
ordering. It was deliberately left unimplemented while the six corrective plans
landed. It is still worth doing — it is just the one that needs a browser in
front of a human.

## Problem

`index.html:485-531` is a hand-built SVG diagram — FIG. 1, the delivery
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
  and may run longer. Total here is 920ms.
- **Function**: an illustrative diagram, not data a user reads to act on.

Passes all four gates — the only opportunity on the page that clears the bar for
a longer, more expressive animation.

## Target

A once-only staggered reveal of the five stages left to right, then the feedback
loop, fired from `IntersectionObserver`.

### Markup

Wrap each stage's existing elements in a `<g class="stage">`, and the feedback
loop's three elements in a `<g class="loop">`. Add `class="schematic-anim"` to
the `<svg>`. **Move no coordinates and change no visual attribute** — wrapping
only.

```html
<!-- target — stage 01, index.html:491-495 wrapped -->
<g class="stage">
  <rect x="14" y="42" width="92" height="80" fill="none" stroke="rgba(198,161,91,0.30)" stroke-width="0.7"></rect>
  <text x="60" y="34" …>01</text>
  <circle cx="60" cy="82" r="5" fill="#D07986"></circle>
  <text x="60" y="66" …>DISCOVER</text>
  <text x="60" y="108" …>ASSESS · NEEDS</text>
</g>
```

Exact blocks to wrap as `<g class="stage">`: `491-495`, `498-502`, `505-509`,
`512-516`, `519-523`. Wrap `526-528` as `<g class="loop">`.

### The pipeline line stays outside the groups — deliberately

The `<line>` at `index.html:488` is **not** wrapped and never fades. It is the
track the stages sit on, so at t=0 the viewer sees the dashed pipeline and the
caption, and the stages populate along it — scaffold first, content second, which
suits the blueprint conceit and means the diagram never looks empty or broken.

The alternative (fade the line in at delay 0 and push the stages to 80ms) was
considered and rejected: the caption below the SVG is outside the `<svg>` and
always visible, so hiding the line only produces a labelled empty box. Do not
"improve" this by wrapping the line.

### CSS

Place next to the existing `.schematic` rules at `index.html:252-256`.

```css
/* target */
/* Hidden only when scripting is available to un-hide it. A no-JS render, or a
   browser without the `scripting` media feature, skips the reveal and shows the
   finished diagram — the reveal is an enhancement, never a precondition for
   seeing it. */
@media (scripting: enabled){
  .schematic-anim .stage,
  .schematic-anim .loop{
    opacity:0;
    transition:opacity 500ms var(--ease-out);
  }
}
.schematic-anim[data-revealed] .stage,
.schematic-anim[data-revealed] .loop{opacity:1;}
/* nth-of-type counts <g> elements: the five stages are g 1-5, the loop is g 6 */
.schematic-anim[data-revealed] .stage:nth-of-type(2){transition-delay:80ms;}
.schematic-anim[data-revealed] .stage:nth-of-type(3){transition-delay:160ms;}
.schematic-anim[data-revealed] .stage:nth-of-type(4){transition-delay:240ms;}
.schematic-anim[data-revealed] .stage:nth-of-type(5){transition-delay:320ms;}
.schematic-anim[data-revealed] .loop{transition-delay:420ms;}
```

80ms per stage — the upper end of the 30–80ms band, chosen because this is
explanatory rather than decorative motion and the eye should follow the sequence.
Total 420ms + 500ms = 920ms.

Because the `opacity:0` starting value comes from the initial style computation
rather than from a JS write, the reveal always has a painted starting value to
transition from — even when the diagram is already in view at load and the
observer callback fires on the first tick.

### JS

**Place this at the very top of the existing `<script>` block (`index.html:875`),
before the nav wiring.** Anything that throws earlier in that script would
otherwise leave the diagram hidden — the one residual failure mode
`@media (scripting: enabled)` cannot cover.

```js
/* target — first statements inside <script> */
// Reveal the FIG. 1 schematic once, when it first scrolls into view. The hidden
// state lives behind @media (scripting: enabled), so this only ever adds the
// reveal — it is never what makes the diagram visible in the first place.
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

Add to the reduced-motion block at the end of the stylesheet (plan 003):

```css
.schematic-anim .stage,
.schematic-anim .loop{opacity:1;transition:none;}
```

This wins on source order over the `@media (scripting: enabled)` rule — both are
specificity `(0,2,0)`, and plan 003's block is last in the sheet. It must stay
last. Reduced motion here means the diagram is simply present; there is no
gentler version of a sequential reveal worth keeping.

## Do NOT animate `stroke-dashoffset` on the pipeline

The obvious instinct is to "draw" the dashed pipeline line (`index.html:488`) and
the feedback loop path (`index.html:526`) with `stroke-dasharray` /
`stroke-dashoffset`. **Both already carry `stroke-dasharray="4 4"` for their
dashed blueprint look.** Overriding `stroke-dasharray` to draw them replaces the
dashes with a solid line; animating `stroke-dashoffset` against the existing
`4 4` makes the dashes *crawl* along the path instead of drawing it. Neither is
the intended effect. Opacity on grouped stages is the correct tool here.

Also do **not** reach for `clip-path: inset()` on the SVG `<g>` elements for a
left-to-right wipe. Percentage `inset()` on SVG child elements resolves
inconsistently across engines and is not worth the risk for this payoff.

## Repo conventions to follow

- The existing `IntersectionObserver` at `index.html:910` is the exemplar for
  observer style, including the explanatory comment above it. Follow its
  formatting — but place the new observer *before* it, at the top of the script,
  for the ordering reason given above.
- SVG stage blocks are already separated by `<!-- stage 0N — NAME -->` comments
  (`index.html:490`, `497`, `504`, `511`, `518`). Keep those comments outside the
  new `<g>` wrappers so the structure stays readable. Comments are not elements
  and do not affect `:nth-of-type` counting.
- Curve token: `var(--ease-out)`, defined at `index.html:52`.
- Component-local media queries are the house style — see `index.html:71`,
  `104`, `122` and `203`.

## Steps

1. Add `class="schematic-anim"` to the `<svg>` at `index.html:486`.
2. Wrap the five stage blocks (`491-495`, `498-502`, `505-509`, `512-516`,
   `519-523`) in `<g class="stage">…</g>`.
3. Wrap the feedback-loop `<path>`, `<polygon>` and `<text>` (`526-528`) in
   `<g class="loop">…</g>`.
4. Add the CSS from Target next to the existing `.schematic` rules
   (`index.html:252-256`).
5. Add the observer JS from Target as the **first statements** inside the
   `<script>` block at `index.html:875`.
6. Add the two-selector reduced-motion override into the plan-003 block at the
   end of the stylesheet, keeping that block last in the sheet.

## Boundaries

- Do NOT change any `x`, `y`, `cx`, `cy`, `r`, `width`, `height`, `d`, `points`,
  `fill`, `stroke`, `stroke-width`, `stroke-dasharray`, `font-size` or
  `letter-spacing` attribute. Wrapping only.
- Do NOT replace `@media (scripting: enabled)` with a bare `opacity:0`, a
  `data-armed` / `js-enabled` attribute set from JS, or a `.no-js` class on
  `<html>` — see the Revision note for why each was rejected.
- Do NOT wrap the pipeline `<line>` at `index.html:488`.
- Do NOT touch the `role="img"` or `aria-label` on the `<svg>`
  (`index.html:486`). Children of a `role="img"` element are already outside the
  accessibility tree, so the cascade is correctly invisible to screen readers —
  removing the role would expose it.
- Do NOT change `.schematic-caption` (`index.html:530`).
- Do NOT animate `stroke-dashoffset` or `clip-path` — see above.
- Do NOT add a motion library. CSS transitions plus one observer is the whole job.
- Do NOT extend this reveal to any other section of the page.
- If the SVG structure has drifted from the line references above, STOP and report.

## Verification

- **Mechanical**:
  - `grep -c 'class="stage"' index.html` returns 5.
  - `grep -c 'class="loop"' index.html` returns 1.
  - `grep -n "stroke-dashoffset" index.html` returns nothing.
  - `grep -c 'stroke-dasharray="4 4"' index.html` still returns 2 — the dashes survive.
  - `grep -n "scripting: enabled" index.html` returns one line.
  - The `.schematic-anim` observer appears in the script *before* `const navOpen`.
- **Feel check** — this plan's payoff cannot be judged from code:
  - Hard-reload with the hero in view. The five stages must appear left to right
    in an even cascade, then the feedback loop last. The rhythm should read as
    "the process runs, then it loops".
  - There must be **no flash** of the complete diagram before the cascade. If you
    see one, the hidden state is being applied after first paint — check it is
    inside the `@media (scripting: enabled)` block and not set from JS.
  - Scroll down past the hero and back up. It must **not** replay.
  - **No-JS check** (the check the first draft of this plan would have failed):
    DevTools → Settings → Debugger → *Disable JavaScript*, then hard-reload. The
    complete diagram must be visible — all five stages and the loop, full
    opacity.
  - **Script-ordering check**: temporarily add `throw new Error('x')` as the
    first line *after* the new observer block, hard-reload, and confirm the
    diagram still reveals. Then move the `throw` to *before* the observer block
    and confirm the diagram stays hidden. That contrast is why the observer must
    run first. Remove the `throw`.
  - Reduced motion: DevTools → Rendering → *Emulate prefers-reduced-motion:
    reduce*, hard-reload. Whole diagram present immediately, no cascade.
  - Throttle to Slow 3G and reload. Confirm the IBM Plex Mono labels do not
    reflow mid-cascade as the webfont swaps in.
  - Look again the next day with fresh eyes. 80ms per stage is a judgement call;
    60ms or 100ms may read better on this diagram than the value specified here.
- **Done when**: the diagram reveals once per page load in sequence, never
  replays, shows in full with JavaScript disabled, shows in full under reduced
  motion, never flashes before revealing, and the dashed blueprint styling of
  both the pipeline and the loop is visually unchanged.
