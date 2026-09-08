# 005 — Gate hover states and kill the scrollbar jump

- **Status**: DONE
- **Commit**: aa7e4d4
- **Severity**: MEDIUM
- **Category**: Accessibility / Cohesion
- **Estimated scope**: 1 file, ~10 lines

## Problem

### A. Four ungated `:hover` rules

```css
/* index.html:113 */  .nav-toggle:hover{border-color:var(--brass);background:rgba(208,121,134,0.10);}
/* index.html:190 */  .btn-primary:hover{background:var(--brass-hover);}
/* index.html:192 */  .btn-ghost:hover{border-color:var(--brass);color:var(--brass-bright);}
/* index.html:334 */  .contact-links a:hover{border-color:var(--brass);color:var(--brass-bright);}
```

A touch device fires a false hover on tap and then leaves it stuck. On a phone,
tapping **Menu** leaves the button holding its brass tint for as long as the page
stays put, which reads as a permanently-active control. These are colour-only,
so this is the milder form of the finding — but it is still an interaction defect
on the majority of this site's likely traffic.

### B. A layout jump running against the panel animation

```js
/* index.html:789 — current */
document.body.style.overflow = open ? 'hidden' : '';
```

Locking scroll removes the scrollbar, so the whole centred 1180px content column
shifts horizontally by the scrollbar's width at the exact moment the nav panel
starts sliding. Two unrelated motions fire together and the page reads as
unstable. Nothing in the animation itself is wrong — the jump is beside it.

## Target

### A. Wrap each hover rule

```css
/* target */
@media (hover: hover) and (pointer: fine){
  .nav-toggle:hover{border-color:var(--brass);background:rgba(208,121,134,0.10);}
}

@media (hover: hover) and (pointer: fine){
  .btn-primary:hover{background:var(--brass-hover);}
  .btn-ghost:hover{border-color:var(--brass);color:var(--brass-bright);}
}

@media (hover: hover) and (pointer: fine){
  .contact-links a:hover{border-color:var(--brass);color:var(--brass-bright);}
}
```

Keep each block next to the component it belongs to rather than collecting them
into one block at the bottom — the stylesheet is organised by component.

### B. Reserve the scrollbar gutter

```css
/* target — index.html:48 */
html{scroll-behavior:smooth;scrollbar-gutter:stable;}
```

One declaration. The gutter is reserved permanently, so removing the scrollbar on
open shifts nothing. This page always overflows the viewport, so the reserved
gutter costs nothing visually.

`:focus-visible` rules (`index.html:114`, `index.html:117`) must **not** be
gated — they are keyboard affordances and have nothing to do with pointers.

## Repo conventions to follow

- The stylesheet already uses component-local media queries for exactly this kind
  of conditional styling — `index.html:64`, `index.html:97`,
  `index.html:159`, `index.html:233-234` are the exemplars. Follow that
  placement, not a global block.
- `README.md:50-55` documents the reasoning style expected for non-obvious
  breakpoints; a one-line comment on the `scrollbar-gutter` declaration is enough.

## Steps

1. Wrap `index.html:113` in `@media (hover: hover) and (pointer: fine){ … }`.
2. Wrap `index.html:190` and `index.html:192` together in one
   `@media (hover: hover) and (pointer: fine){ … }` block — they are adjacent
   and belong to the same component.
3. Wrap `index.html:334` in `@media (hover: hover) and (pointer: fine){ … }`.
4. Add `scrollbar-gutter:stable;` to the `html` rule at `index.html:48`, with a
   short comment noting it prevents the content shift when the nav panel locks
   scroll.

## Boundaries

- Do NOT gate `:focus-visible` (`index.html:114`, `index.html:117`) or the
  `.skip-link:focus` rule (`index.html:76`).
- Do NOT gate `:active` rules — a press is a real press on touch.
- Do NOT change any colour value, border, or background inside the hover rules.
  This plan only wraps them.
- Do NOT replace the `body.style.overflow` scroll lock with a JS scrollbar-width
  calculation or a padding compensation hack. `scrollbar-gutter` is the fix.
- Do NOT touch `overflow-x:hidden` on `body` (`index.html:54`).

## Verification

- **Mechanical**: `grep -c "hover: hover" index.html` returns 3.
  `grep -n "scrollbar-gutter" index.html` returns one line inside the `html` rule.
- **Feel check**:
  - Desktop: hover each of Menu, both hero CTAs, both contact links. All four
    hover states must still work exactly as before.
  - DevTools → device emulation (a phone preset, touch input): tap **Menu**, then
    close the panel. The Menu button must return to its untinted resting state —
    no stuck brass background.
  - Click **Menu** on desktop with a visible scrollbar and watch the top bar
    wordmark. It must not shift horizontally as the panel slides in. Before the
    fix it jumps by the scrollbar width.
  - Keyboard: Tab to a hero CTA. The focus ring must still appear (proving
    `:focus-visible` was not gated).
- **Done when**: hover works on pointer devices, does not stick on touch, focus
  rings are unaffected, and opening the panel shifts nothing horizontally.
