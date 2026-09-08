# 006 — Stagger the nav panel links and add a hover affordance

- **Status**: DONE
- **Commit**: aa7e4d4
- **Severity**: MEDIUM
- **Category**: Cohesion / Missed opportunities (additive)
- **Estimated scope**: 1 file, ~14 lines

## Problem

```css
/* index.html:131-137 — current */
.nav-panel a{
  font-family:'Fraunces',serif;font-size:clamp(28px,7vw,60px);font-weight:500;
  display:flex;align-items:baseline;gap:18px;
  padding:14px 0;border-bottom:1px solid var(--rule);
  color:var(--text-light);
}
.nav-panel a span{font-family:'IBM Plex Mono',monospace;font-size:14px;color:var(--brass-bright);}
```

Two gaps:

1. **Five 60px links arrive as one solid block.** The panel is a rare/occasional
   surface on a marketing page — the tier where a group entrance is permitted and
   where a 30–80ms stagger belongs. Everything-at-once here is a cohesion finding.
2. **No hover state at all.** Five large links, and moving a pointer over them
   produces nothing. Every other control on the page has a hover affordance.

## Gate result

- **Frequency**: occasional — a visitor opens this menu zero to two times.
  Eligible for standard animation, and the delight budget is available.
- **Purpose**: **Preventing a jarring change** (the block of links no longer
  lands all at once) plus **Feedback** for the hover and press states.
- **Function**: navigation controls, not data being read.

## Target

Stagger with `transition-delay`, **not** `@keyframes` and `animation-delay`. The
panel is a toggle: keyframes restart from zero when the user reopens mid-close,
transitions retarget from wherever the links currently are.

```css
/* target */
.nav-panel a{
  font-family:'Fraunces',serif;font-size:clamp(28px,7vw,60px);font-weight:500;
  display:flex;align-items:baseline;gap:18px;
  padding:14px 0;border-bottom:1px solid var(--rule);
  color:var(--text-light);
  opacity:0;transform:translateY(8px);
  transition:opacity 300ms var(--ease-out), transform 300ms var(--ease-out),
             color 200ms ease;
}
.nav-panel.open a{opacity:1;transform:translateY(0);}
.nav-panel.open a:nth-of-type(2){transition-delay:40ms;}
.nav-panel.open a:nth-of-type(3){transition-delay:80ms;}
.nav-panel.open a:nth-of-type(4){transition-delay:120ms;}
.nav-panel.open a:nth-of-type(5){transition-delay:160ms;}
.nav-panel.open a:active{transform:scale(0.97);transition-duration:160ms;transition-delay:0ms;}

@media (hover: hover) and (pointer: fine){
  .nav-panel a:hover{color:var(--brass-bright);}
}
```

Four details the executor must not "tidy":

- **`:nth-of-type`, not `:nth-child`.** `.nav-close` is the panel's first child
  element (`index.html:365`), so `:nth-child` would be off by one.
  `:nth-of-type` counts only the `a` elements, so the five links are 1–5 and the
  first correctly takes no delay.
- **Delays live on `.nav-panel.open`, not on the base rule.** On close the base
  rule applies with no delay, so the links leave together with the panel instead
  of staggering out. Exit does not need its own choreography — the panel is
  already carrying it off screen.
- **The last delay is 160ms + 300ms = 460ms**, which lands with the panel's 450ms
  open from plan 002. The stagger must finish with the panel, never after it.
- **`.nav-panel.open a:active` has specificity `(0,3,1)`**, matching the
  `:nth-of-type` rules, and must be declared **after** them so the press
  overrides both the entrance's slower 300ms transform channel and any stagger
  delay. A plain `.nav-panel a:active` is `(0,2,1)` and would lose — a press on
  the fourth link would then wait 120ms before responding.

Stagger is decorative: the links are already clickable while it plays, and
nothing blocks interaction.

## Repo conventions to follow

- Curve token from plan 001: `var(--ease-out)`. `ease` on the colour channel,
  matching every other hover transition in the file.
- Hover gating follows plan 005's pattern — a component-local
  `@media (hover: hover) and (pointer: fine)` block placed next to the rule.
- `--brass-bright` is the established interactive accent on dark backgrounds
  (`index.html:104`, `index.html:137`, `index.html:192`) and is documented at
  `index.html:38` as AA-compliant on `--ink` at small sizes. Use it, do not
  introduce a new colour.

## Steps

1. Apply plans 001 and 002 first — this plan needs `--ease-out` and is timed
   against plan 002's 450ms open.
2. In `.nav-panel a` (`index.html:131-136`), keep every existing declaration and
   append `opacity:0;`, `transform:translateY(8px);` and the three-channel
   `transition` from Target.
3. Immediately after, add the `.nav-panel.open a` rule setting `opacity:1` and
   `transform:translateY(0)`.
4. Add the four `:nth-of-type(2..5)` `transition-delay` rules, in order.
5. Add the `.nav-panel.open a:active` rule **after** those four.
6. Add the gated `:hover` block.
7. Leave `.nav-panel a span` (`index.html:137`) untouched.

## Boundaries

- Do NOT use `@keyframes` / `animation-delay` — see Target.
- Do NOT stagger the exit.
- Do NOT stagger any other group on the page. `.approach-grid` (6 cards),
  `.cert-grid` (10 cards), `.cap-grid` (6 columns), `.timeline` (7 rows) and
  `.work-item` (6 entries) were all considered and rejected: they are content a
  reader is there to read, not an occasional surface, and revealing them would
  add six waits to a static spec sheet.
- Do NOT add a `translateX` slide or an underline animation to the hover — colour
  only. The page's personality is a crisp blueprint, not a playful consumer app.
- Do NOT change the link markup, order, or the `<span>` sheet numbers.
- Do NOT add movement to `:hover`; that would need pointer gating *and* a
  reduced-motion branch for no gain.

## Verification

- **Mechanical**: `grep -n "nth-of-type" index.html` shows five rules — four
  delays plus the reduced-motion reset from plan 003.
  `grep -n "animation-delay" index.html` returns nothing.
- **Feel check**: open `index.html`.
  - Click **Menu**. The five links must arrive top-to-bottom in a visible but
    quick cascade, and the last one must settle as the panel finishes — not after.
  - Click **Close**. All five must leave together with the panel. If they cascade
    out, the delays were put on the base rule instead of `.open`.
  - **Interruption test**: click Menu then Close before the cascade finishes. The
    links must fade from wherever they are, never snap to invisible and restart.
  - Press and hold the **fourth** link (Capabilities). It must compress
    immediately, with no delay. A perceptible wait means the `:active` rule is
    losing to the `:nth-of-type` delay — check its position and specificity.
  - Hover each link on desktop: colour shifts to brass. On touch emulation, tap
    a link and confirm the colour does not stick.
  - In DevTools → Animations at 10%, confirm the gap between consecutive links is
    even and that no link starts before the one above it.
- **Done when**: the cascade is 40ms per step, finishes with the panel, reverses
  cleanly mid-flight, and a press on any link responds instantly.
