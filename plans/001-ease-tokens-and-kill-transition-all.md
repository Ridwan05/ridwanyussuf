# 001 — Add easing tokens and replace `transition: all`

- **Status**: DONE
- **Commit**: aa7e4d4
- **Severity**: HIGH
- **Category**: Easing & duration / Performance
- **Estimated scope**: 1 file, ~12 lines

## Problem

Two rules animate `all`, which animates every property that ever changes on the
element — including properties that are not GPU-composited — and hides which
properties the author actually meant:

```css
/* index.html:187 — current */
.btn{
  font-family:'IBM Plex Mono',monospace;font-size:13px;letter-spacing:.05em;
  padding:14px 26px;border-radius:2px;transition:all .2s;display:inline-block;
}
```

```css
/* index.html:332 — current */
.contact-links a{
  font-family:'IBM Plex Mono',monospace;font-size:13px;
  border:1px solid var(--rule);padding:14px 24px;border-radius:2px;transition:all .2s;
}
```

Separately, the file has no easing vocabulary. The only custom curve in the page
is hand-typed inline at `index.html:128`. Every other transition falls back to
the browser's built-in `ease`, which is too weak for deliberate motion.

## Target

Add the shared curve tokens to the existing `:root` block, then name the exact
properties in both transitions.

```css
/* target — added to the :root block at index.html:31 */
--ease-out: cubic-bezier(0.23, 1, 0.32, 1);
--ease-drawer: cubic-bezier(0.32, 0.72, 0, 1);
```

Only these two. The standard vocabulary also includes
`--ease-in-out: cubic-bezier(0.77, 0, 0.175, 1)` for elements that move or morph
on screen, but nothing on this page does — adding it would ship an unused
declaration. Add it at the point something needs it, not before.

```css
/* target — index.html .btn */
transition:background 200ms ease, border-color 200ms ease, color 200ms ease,
           transform 160ms var(--ease-out);
```

```css
/* target — index.html .contact-links a */
transition:border-color 200ms ease, color 200ms ease,
           transform 160ms var(--ease-out);
```

`ease` is correct on the colour channels — the easing decision table assigns
`ease` to hover and colour changes. The `transform` channel is the press
feedback added by plan 004; include it here so the property list is written once.

## Repo conventions to follow

- All tokens live in the single `:root` block at `index.html:31-46`, one per
  line, with an inline `/* … */` comment where the value needs justifying —
  see `--brass-hover` at `index.html:37` and `--brass-bright` at `index.html:38`
  for the exemplar comment style.
- `README.md:71` documents that colour tokens live in `:root`. Curves go in the
  same block.
- The file uses no build step and no preprocessor. Plain CSS custom properties only.

## Steps

1. In the `:root` block at `index.html:31`, after `--rule-dark`, add the two
   `--ease-*` declarations from Target above, preceded by a
   `/* ---------- motion ---------- */`-style comment consistent with the
   section comments already used in the stylesheet.
2. In `.btn` (`index.html:185-188`), delete `transition:all .2s;` and add the
   named `transition` from Target.
3. In `.contact-links a` (`index.html:330-333`), delete `transition:all .2s;`
   and add the named `transition` from Target.

## Boundaries

- Do NOT touch markup, copy, colour values, or layout.
- Do NOT add a duration-scale token system — six transitions do not justify one.
- Do NOT change `.nav-toggle`'s transition here; it is already property-named.
  Plan 004 extends it.
- Do NOT add dependencies.
- If `transition:all` no longer appears at those lines, STOP and report.

## Verification

- **Mechanical**: `grep -n "transition:all" index.html` returns nothing.
  `grep -n "\-\-ease-" index.html` shows the two tokens defined in `:root`.
- **Feel check**: open `index.html`, hover a hero CTA and a contact link. The
  colour/border change should look identical to before — this plan is a
  correctness fix, not a visible change. Nothing should move on hover.
- **Done when**: no `transition:all` in the file, three `--ease-*` tokens exist,
  and hover on `.btn` / `.contact-links a` looks unchanged.
