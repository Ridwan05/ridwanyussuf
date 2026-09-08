# 004 — Add press feedback to every pressable element

- **Status**: DONE
- **Commit**: aa7e4d4
- **Severity**: MEDIUM
- **Category**: Missed opportunities (additive)
- **Estimated scope**: 1 file, ~10 lines

## Problem

`grep -n ":active" index.html` at commit `aa7e4d4` returns **nothing**. Every
pressable element on the page — hero CTAs, the Menu button, the Close button, the
five nav links, the two contact links — responds to a press with no acknowledgement
at all. On touch, where there is no hover state to fall back on, a tap produces
nothing until the page navigates.

The affected declarations:

```css
/* index.html:185-188 */   .btn{ …no :active… }
/* index.html:105-112 */   .nav-toggle{ …no :active… }
/* index.html:138-143 */   .nav-close{ …no transition, no :active… }
/* index.html:131-136 */   .nav-panel a{ …no :active, no hover at all… }
/* index.html:330-333 */   .contact-links a{ …no :active… }
```

## Gate result

- **Frequency**: occasional to tens-per-visit. That tier permits
  near-imperceptible motion only — which is exactly what a 0.97 scale at 160ms is.
- **Purpose**: **Feedback** — confirming the interface heard the user.
- **Function**: these are controls, not data being read. Motion helps.

## Target

`transform: scale(0.97)` on `:active`, `transition: transform 160ms var(--ease-out)`.
Subtle by design — the sanctioned range is 0.95–0.98.

```css
/* target */
.btn:active{transform:scale(0.97);}
.nav-toggle:active{transform:scale(0.97);}
.nav-close:active{transform:scale(0.97);}
.contact-links a:active{transform:scale(0.97);}
```

Each element's existing `transition` must gain the transform channel:

```css
/* target — .btn */
transition:background 200ms ease, border-color 200ms ease, color 200ms ease,
           transform 160ms var(--ease-out);

/* target — .nav-toggle */
transition:border-color 200ms ease, background 200ms ease,
           transform 160ms var(--ease-out);

/* target — .nav-close (has no transition today) */
transition:border-color 200ms ease, transform 160ms var(--ease-out);

/* target — .contact-links a */
transition:border-color 200ms ease, color 200ms ease,
           transform 160ms var(--ease-out);
```

`scale()` scales children too, so the label comes along — that is what makes it
read as a physical press rather than a resize.

The nav panel links are handled in plan 006, because their `transform` channel is
shared with the stagger entrance and needs a duration and delay override.

**No hover gating is required for `:active`** — a press is a real press on touch.
Hover gating is plan 005 and is a separate concern.

## Repo conventions to follow

- Curve token from plan 001: `var(--ease-out)`. Never re-type the cubic-bezier.
- The stylesheet groups a component's states immediately after its base rule —
  `.btn` / `.btn-primary` / `.btn-primary:hover` at `index.html:185-192` is the
  exemplar. Put each `:active` rule directly after the base rule it belongs to.

## Steps

1. Apply plan 001 first — this plan requires `--ease-out`.
2. `.btn` (`index.html:185-188`): set the four-channel `transition` from Target,
   then add `.btn:active{transform:scale(0.97);}` on the following line.
3. `.nav-toggle` (`index.html:105-112`): append
   `transform 160ms var(--ease-out)` to its existing `transition`, then add
   `.nav-toggle:active{transform:scale(0.97);}`.
4. `.nav-close` (`index.html:138-143`): add the `transition` from Target (it has
   none today), then add `.nav-close:active{transform:scale(0.97);}`.
5. `.contact-links a` (`index.html:330-333`): set the three-channel `transition`
   from Target, then add `.contact-links a:active{transform:scale(0.97);}`.

## Boundaries

- Do NOT exceed the 0.95–0.98 scale range, and do NOT add a scale to `:hover` —
  hover motion on a page this dense is noise, and hover would need pointer gating.
- Do NOT add press feedback to `.skip-link` (`index.html:70-76`). It is
  keyboard-initiated and must be instant.
- Do NOT add press feedback to non-interactive elements — `.stat`, `.cert`,
  `.work-item`, `.approach-card`, `.cap-col li` are content, not controls.
- Do NOT animate `width`, `height`, `padding` or `box-shadow` for the press.
- Do NOT touch the nav panel links here — plan 006 owns them.
- Do NOT add dependencies.

## Verification

- **Mechanical**: `grep -c ":active" index.html` returns at least 4.
  `grep -n "scale(0" index.html` shows only `scale(0.97)` values — no `scale(0)`.
- **Feel check**: open `index.html`.
  - Press and hold each of: **View Selected Work**, **Get in Touch**, **Menu**,
    **Close**, the email link, the LinkedIn link. Each must visibly compress
    slightly and spring back on release.
  - The compression must be barely perceptible in normal use. If it reads as a
    "bounce" or a "shrink", the scale is too aggressive — check it is 0.97.
  - In DevTools → Animations at 10% playback, confirm the scale is fastest at the
    start of the release (that is `--ease-out` working) and that nothing else on
    the element moves.
  - On a touch device or DevTools device emulation, tap each control and confirm
    the press registers.
- **Done when**: all six controls acknowledge a press, at 0.97 scale over 160ms,
  and no non-interactive element gained an `:active` state.
