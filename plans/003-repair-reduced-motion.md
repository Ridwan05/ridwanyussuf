# 003 — Repair the reduced-motion block

- **Status**: DONE
- **Commit**: aa7e4d4
- **Severity**: HIGH
- **Category**: Accessibility
- **Estimated scope**: 1 file, ~25 lines (one block deleted, one added)

## Problem

```css
/* index.html:56-59 — current */
@media (prefers-reduced-motion: reduce){
  html{scroll-behavior:auto;}
  *{animation-duration:0.01ms !important; animation-iteration-count:1 !important; transition-duration:0.01ms !important;}
}
```

This is the reduced-motion sledgehammer. `*` plus `!important` collapses **every**
transition on the page to 0.01ms — including the colour and opacity feedback that
tells a user the interface heard them. Reduced motion means *fewer and gentler*
animations, not zero: keep the transitions that aid comprehension, remove the
movement and position changes.

There is also a **source-order defect** that makes any targeted fix silently fail.
The block sits at line 56, before `.nav-panel` (line 128), `.btn` (line 187) and
`.contact-links a` (line 332) are declared. Equal-specificity rules declared
later win, so a targeted reduced-motion rule written at line 56 would be
overridden by the very rules it is meant to override. That is why the current
code needs `*` and `!important` to work at all.

## Target

Delete the block at `index.html:56-59` entirely and add this at the **end** of
the `<style>` element, after the `::selection` rule (`index.html:350`), so it
wins on source order without `!important`:

```css
/* target — placed last in <style> */
/* ---------- reduced motion ---------- */
/* Last in the sheet so these rules win on source order without !important.
   Reduced motion means fewer and gentler animations, not zero: movement
   (transform) is dropped, the colour and opacity feedback that confirms the
   interface heard you is kept. */
@media (prefers-reduced-motion: reduce){
  html{scroll-behavior:auto;}

  /* the panel crossfades in place instead of sliding down */
  .nav-panel,
  .nav-panel.open{
    transform:none;
    transition:opacity 200ms ease, visibility 200ms;
  }
  .nav-panel{opacity:0;}
  .nav-panel.open{opacity:1;}

  /* links arrive with the panel — no travel, no stagger */
  .nav-panel a,
  .nav-panel.open a:nth-of-type(n){
    transform:none;
    transition:opacity 200ms ease, color 200ms ease;
    transition-delay:0ms;
  }

  /* press feedback keeps its colour channel, loses the squash */
  .btn:active,
  .nav-toggle:active,
  .nav-close:active,
  .nav-panel.open a:active,
  .contact-links a:active{transform:none;}
}
```

Two specificity notes the executor must not "simplify":

- `.nav-panel.open a:nth-of-type(n)` is written with `:nth-of-type(n)` — which
  matches every `a` — purely to reach specificity `(0,3,1)`, matching plan 006's
  `.nav-panel.open a:nth-of-type(2..5)` stagger-delay rules. A plain
  `.nav-panel.open a` is `(0,2,1)` and would lose to them, leaving the stagger
  delays active under reduced motion.
- `.nav-panel.open a:active` is `(0,3,1)` for the same reason.

Because `transform:none` is applied to the closed `.nav-panel`, the closed panel
sits at `translateY(0)` covering the viewport — but it stays `visibility:hidden`,
`opacity:0` and `inert`, so it receives no pointer events and stays out of the
accessibility tree. This is intentional.

## Repo conventions to follow

- Stylesheet sections are introduced by a
  `/* ---------- section name ---------- */` banner comment — see
  `index.html:80`, `index.html:90`, `index.html:145`, `index.html:161`.
  Use the same form.
- Non-obvious decisions get a `/* … */` rationale directly above them, as at
  `index.html:125-126`, `index.html:155-158`, `index.html:306-307`.

## Steps

1. Apply plans 002, 004 and 006 first. This plan overrides their rules by source
   order, so it must be the last of the four to land.
2. Delete lines 56-59 (`@media (prefers-reduced-motion: reduce){ … }`) from
   `index.html`. Leave the surrounding `body{}` and `a{}` rules untouched.
3. Insert the Target block immediately after the `::selection` rule at
   `index.html:350`, still inside `<style>`.

## Boundaries

- Do NOT keep a `*{transition-duration:0.01ms !important}` fallback alongside the
  new block — it would defeat the entire plan.
- Do NOT remove `html{scroll-behavior:auto;}`; smooth scroll is movement and must
  still be dropped.
- Do NOT touch the `visibility` mechanics or the `inert` attribute handling.
- Do NOT add `prefers-reduced-motion` handling for animations that do not exist
  yet (plan 007 ships its own).
- If the block at 56-59 does not match the Problem excerpt, STOP and report.

## Verification

- **Mechanical**: `grep -n "0.01ms" index.html` returns nothing.
  `grep -n "prefers-reduced-motion" index.html` returns exactly one line, and it
  is after the `::selection` rule.
- **Feel check**: DevTools → Rendering → *Emulate CSS prefers-reduced-motion:
  reduce*, then:
  - Click **Menu**. The panel must **crossfade** in place — no vertical slide —
    and the links must all arrive together, not staggered.
  - Press and hold a hero CTA. It must **not** scale, but the background colour
    change must still happen at a visible, gentle speed. If the colour snaps
    instantly, the sledgehammer is still in effect somewhere.
  - Click a nav link. Anchor navigation must jump, not smooth-scroll.
  - Tab through the page with the panel closed. Focus must never land on a nav
    panel link.
- **Done when**: with reduced motion on, no element translates or scales, colour
  and opacity transitions still run at ~200ms, and the closed panel is still
  unreachable by keyboard.
