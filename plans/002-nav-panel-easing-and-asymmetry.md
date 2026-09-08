# 002 — Fix nav panel easing and make enter/exit asymmetric

- **Status**: DONE
- **Commit**: aa7e4d4
- **Severity**: HIGH
- **Category**: Easing & duration / Interruptibility
- **Estimated scope**: 1 file, ~8 lines

## Problem

The full-screen nav panel is the single most prominent interaction on the site,
and it starts slow in both directions:

```css
/* index.html:119-130 — current */
.nav-panel{
  position:fixed;inset:0;z-index:60;
  background:var(--ink);
  display:flex;flex-direction:column;justify-content:center;
  padding:40px;
  transform:translateY(-100%);
  visibility:hidden;
  transition:transform .45s cubic-bezier(.65,0,.35,1), visibility .45s;
}
.nav-panel.open{transform:translateY(0);visibility:visible;}
```

Two defects:

1. **`cubic-bezier(.65,0,.35,1)` is a symmetric ease-in-out.** Its first control
   point `(.65, 0)` holds the curve flat off the start, so the panel eases *in*.
   The easing decision table assigns `ease-out` to anything entering or exiting;
   `ease-in` on UI delays the exact moment the user is watching. `ease-out` at
   200ms feels faster than `ease-in` at 200ms. This is the finding that makes the
   menu feel heavy.
2. **Enter and exit are both 450ms.** Opening is the deliberate phase — the user
   asked to see the menu and it should read as a surface arriving. Closing is the
   system getting out of the way and should snap. Symmetric timing on a
   press-and-release style interaction is a finding.

`visibility` being transitioned alongside `transform` is **correct and
deliberate** — the comment at `index.html:125-126` documents it, and it is what
keeps the closed panel out of the tab order and the accessibility tree. Do not
change it, only keep its duration matched to the transform in each state.

## Target

```css
/* target */
.nav-panel{
  /* …unchanged layout properties… */
  transform:translateY(-100%);
  visibility:hidden;
  transition:transform 300ms var(--ease-drawer), visibility 300ms;
}
.nav-panel.open{
  transform:translateY(0);visibility:visible;
  transition:transform 450ms var(--ease-drawer), visibility 450ms;
}
```

`--ease-drawer: cubic-bezier(0.32, 0.72, 0, 1)` is the iOS drawer curve and is
the sanctioned token for exactly this component class.

The transition that runs is the one declared on the **destination** state, so:

- base rule → the transition *to* closed → **300ms** (close, snappy)
- `.open` rule → the transition *to* open → **450ms** (open, deliberate)

Both sit inside the 200–500ms modal/drawer budget.

## Repo conventions to follow

- Curve tokens come from plan 001's `:root` block. Do not re-type the
  cubic-bezier inline — `var(--ease-drawer)` only.
- The stylesheet documents non-obvious motion decisions in a `/* … */` comment
  directly above the declaration. `index.html:125-126` and `index.html:155-158`
  are the exemplars. Add one explaining the asymmetry and the curve swap.

## Steps

1. Apply plan 001 first — this plan requires `--ease-drawer`.
2. In `.nav-panel` (`index.html:119-129`), replace the
   `transition:transform .45s cubic-bezier(.65,0,.35,1), visibility .45s;` line
   with `transition:transform 300ms var(--ease-drawer), visibility 300ms;`.
3. Above it, add a comment recording that close is 300ms, open is 450ms on the
   `.open` rule, and that the previous curve eased in.
4. In `.nav-panel.open` (`index.html:130`), append
   `transition:transform 450ms var(--ease-drawer), visibility 450ms;`.

## Boundaries

- Do NOT remove the `visibility` transition or the `inert` attribute handling —
  both are deliberate accessibility mechanics documented in `README.md:67-70`.
- Do NOT change the slide direction. The panel enters from the top edge, which
  is spatially consistent with the Menu button in the top bar.
- Do NOT convert this to `@keyframes` — it is a toggle, and transitions retarget
  from the current position when the user reverses mid-slide.
- Do NOT touch the JS in `index.html:780-800`.
- If the current transition line does not match the Problem excerpt, STOP.

## Verification

- **Mechanical**: `grep -n "cubic-bezier(.65,0,.35,1)" index.html` returns
  nothing. `grep -n "ease-drawer" index.html` shows the token used twice.
- **Feel check**: open `index.html` in a browser.
  - Click **Menu**. The panel should leave the top edge immediately at speed and
    settle, rather than creeping off the mark.
  - Click **Close**. It should clear the viewport noticeably faster than it
    arrived.
  - **Interruption test**: click Menu and click Close while the panel is still
    sliding down. It must reverse from wherever it currently is, never jump back
    to the top and restart.
  - In DevTools → Animations, set playback to 10% and confirm the panel is moving
    fastest in the first third of the open, not the middle.
- **Done when**: the curve is `var(--ease-drawer)` in both states, close is
  300ms, open is 450ms, and a mid-slide reversal is smooth.
