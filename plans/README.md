# Animation plans

Produced by `improve-animations` (Emil Kowalski skill set) against commit `aa7e4d4`.

The site is a single-file static page — every plan edits `index.html` and nothing
else. There is no build step, no motion library, and no dependency to add.

## Plans

| # | Title | Severity | Category | Status |
|---|-------|----------|----------|--------|
| [001](001-ease-tokens-and-kill-transition-all.md) | Add easing tokens, replace `transition: all` | HIGH | Easing & duration / Performance | DONE |
| [002](002-nav-panel-easing-and-asymmetry.md) | Fix nav panel easing, make enter/exit asymmetric | HIGH | Easing & duration / Interruptibility | DONE |
| [003](003-repair-reduced-motion.md) | Repair the reduced-motion block | HIGH | Accessibility | DONE |
| [004](004-press-feedback.md) | Add press feedback to every pressable element | MEDIUM | Missed opportunities | DONE |
| [005](005-gate-hover-and-scrollbar-jump.md) | Gate hover states, kill the scrollbar jump | MEDIUM | Accessibility / Cohesion | DONE |
| [006](006-nav-link-stagger.md) | Stagger nav panel links, add hover affordance | MEDIUM | Cohesion / Missed opportunities | DONE |
| [007](007-schematic-reveal.md) | Reveal the FIG. 1 schematic on first view | LOW | Missed opportunities | TODO |

## Execution order

001 first — it introduces the `--ease-*` tokens that 002, 004, and 006 reference.
003 must land after 002, 004, and 006, because the reduced-motion block overrides
their rules by source order and has to sit at the end of the stylesheet.

```
001 ──► 002 ──┐
        004 ──┼──► 003 ──► 005
        006 ──┘     │
                    └───► 007
```

## Dependencies

- **002, 004, 006 require 001** — they use `var(--ease-out)` / `var(--ease-drawer)`.
- **003 must be applied last of 002/004/006** — it is a source-order override.
- **007 requires 001 and 003.** It uses `var(--ease-out)`, and its reduced-motion
  override goes into 003's block and depends on that block staying last in the
  stylesheet to beat 007's own `@media (scripting: enabled)` rule on source
  order. Both are specificity `(0,2,0)`, so position is the only thing deciding.

## Revisions

- **007 revised 2026-09-08** (against `b90a6de`). The original draft hid the
  schematic with a plain `opacity:0` in CSS, which would have made the hero
  diagram invisible whenever JS did not run. It now hides behind
  `@media (scripting: enabled)`. A `data-armed` attribute set from JS was
  considered as the fix and rejected — it trades the blank diagram for a flash of
  the complete diagram before the cascade. See the plan's Revision note.

## Not planned (deliberately rejected)

Recorded so nobody re-proposes them. Full reasoning in the audit.

- Hero entrance / staggered headline — delays the copy a reader is there to read.
- Scroll reveal on all six sections — this page is a static spec sheet by design.
- Animating the fixed title-block label swap — fires continuously while scrolling.
- Animating the skip link — keyboard-initiated, must be instant.
- Animating the `.grid-bg` schematic grid — decoration on every screen, always visible.
