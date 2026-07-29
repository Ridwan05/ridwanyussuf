# Ridwan Yussuf — personal site

A single-page portfolio for Ridwan Yussuf, IT Manager & Digital Transformation
Expert (Abuja, Nigeria). Laid out as a blueprint-style spec sheet: a fixed
schematic grid, per-section sheet numbering, and an SVG delivery-lifecycle
diagram.

## Structure

The entire site is one file:

```
index.html    markup, CSS and JS — no build step, no dependencies
README.md
```

CSS and JavaScript are inlined in `index.html`. The favicon is an inline SVG
data URI. The only external request is the Google Fonts stylesheet for
Fraunces, IBM Plex Mono and Inter — everything else works offline (with
fallback system fonts).

## Running it locally

Open `index.html` in a browser. Nothing to install or compile.

If you'd rather serve it over HTTP — worth doing before deploying, since it
matches how the site will actually load:

```sh
python -m http.server 8000    # then visit http://localhost:8000
```

## Sections

| Sheet | Section | Anchor |
|-------|---------|--------|
| 01 | Hero | `#top` |
| 02 | Approach — process cards and leadership triptych | `#approach` |
| 03 | Selected Work — five case entries with stats | `#work` |
| 04 | Experience — role timeline, certifications and education | `#experience` |
| 05 | Capabilities — six skill columns | `#capabilities` |
| — | Contact | `#contact` |

Navigation is a full-screen overlay panel opened from the `Menu` button.

## Notes for editing

A few behaviours are less obvious than they look, and are easy to break:

- **The fixed sheet title-block** (bottom left) is deliberately hidden below a
  `1639px` viewport. It is 221px wide at `left:24px`, so its right edge sits at
  245px, while the centred 1180px content column begins at
  `(100vw - 1180) / 2 + 40`. Those only clear each other at roughly 1620px;
  below that the block sat directly on top of section body copy. Widening the
  block, or lowering that breakpoint, reintroduces the overlap.
- **The title-block label** is driven by an `IntersectionObserver` using
  `rootMargin: '-45% 0px -55% 0px'` — a thin band at 45% viewport height. It
  deliberately does *not* use a ratio threshold: a section taller than twice
  the viewport can never reach `threshold: 0.5`, which left the label stuck on
  the hero permanently.
- **Anchor targets** rely on `section { scroll-margin-top: 96px }` to clear the
  79px fixed topbar. Changing the topbar's height means changing this too.
- **The certification grid** draws its hairlines with per-cell borders rather
  than a `gap` over a tinted container background. With 10 cards in 4 columns
  the final row is ragged, and the container-background approach painted a
  visible tinted block in the two empty cells.
- **The nav panel** is `visibility:hidden` and `inert` when closed, so its
  links stay out of the tab order and the accessibility tree. It also closes on
  `Escape` and returns focus to the toggle. Hiding it with `transform` alone
  leaves five invisible links keyboard-reachable.
- **Colour tokens** live in `:root`. The accent (`--brass-bright`) and the
  primary-button pairing were chosen for contrast, not just hue — see below
  before changing them.

## Accessibility

- All 37 text/background pairings in the page meet WCAG 2.1 AA (4.5:1 for body
  text, 3:1 for large text), verified by computing relative luminance for each
  pairing rather than by eye.
- `--brass-bright` is `#D07986` specifically so it clears 4.5:1 against the ink
  background at 11–14px. The earlier `#C15F6B` was 4.33:1.
- `.btn-primary` uses light text on `--brass` (6.6:1). Dark text on that
  background is 2.2:1 and fails badly.
- Focus indicators are darkened to `--brass` inside light `.sheet` sections,
  where the bright accent is only 2.6:1 against paper.
- A skip link precedes the topbar, and `prefers-reduced-motion` is honoured.

## Deployment

Any static host will serve this as-is. For GitHub Pages, enable it for the
`main` branch in repository settings; the site will be served from
`ridwan05.github.io/ridwanyussuf/` unless the repository is renamed to
`Ridwan05.github.io`.

### Before going live

`index.html` carries an open TODO in its `<head>`: `og:url` and `og:image` are
not set, because both need a final domain. Until an `og:image` (1200×630 PNG at
an absolute URL) exists, links shared to LinkedIn or Slack render as a bare
text card — which is the single most visible thing still missing.
