# The performance floor

Smoothness is mostly the absence of jank. These rules remove the common sources.

## What you may animate

The compositor can move a layer without repainting it when you animate `transform`,
`opacity`, and (usually) `filter`. Everything else forces layout or paint on the main
thread, which is where 60fps dies.

Sanctioned exceptions, each defensible:

| Property | Where | Why it is OK |
|---|---|---|
| `grid-template-rows` | Accordions | Content-height animation with no JS measuring; contained reflow |
| `border-radius` | Button morph | Tiny element, paint-only |
| `width` | Fill circles, indicator dots | Tiny elements |
| `box-shadow` | Card hover | Short, on hover only, small blur radii |

If a property is not `transform`/`opacity`/`filter` and not on this list, justify it or
find another way.

## will-change

Use it in the two or three places where dropped frames were actually observed: hero
word spans during entrance, 3D carousel cards during drag. Do not spray it; every
`will-change` pins a compositor layer and costs memory, and browsers deprioritize it
when overused.

## Event hygiene

- Scroll and touch listeners: `{ passive: true }`, always.
- One capture-phase listener on `document` can serve overlays regardless of which
  element actually scrolls.
- Skip throttle/debounce when the handler is trivial (integer compares and a no-op
  state set). The wrapper can cost more than the work.
- `matchMedia("...").addEventListener("change", ...)` instead of resize listeners.
- `ResizeObserver` instead of measuring on window resize.
- Hand-rolled physics: one requestAnimationFrame loop, cancelled when idle, writing
  `element.style.transform` directly, never through framework state per frame.

## Layout shift: zero, from anything

- Explicit `width`/`height` attributes or an `aspect-ratio` wrapper on every image,
  video, and embed. No exceptions.
- `text-wrap: balance` on headings; `min-height` floors on cards whose content varies.
- Fluid type and spacing via `clamp()` so nothing rewraps or jumps at breakpoints:
  `font-size: clamp(40px, 6.4vw, 96px)` for a hero H1,
  `padding-block: clamp(56px, 9vw, 130px)` for sections.
- Grid safety: `repeat(auto-fit, minmax(min(100%, 300px), 1fr))`. The inner `min()`
  prevents overflow on 320px screens.

## Fonts

- Self-host everything as woff2. No third-party font connections.
- `font-display: swap`.
- Declare the face with `font-weight: 100 900` even if you own one weight file. This
  stops the browser synthesizing faux bold, which changes glyph widths and reflows
  every heading after swap.
- Preload the LCP-critical font or poster image from the HTML head; skip preloading
  everything else.

## Loading

- Hero image/poster: `fetchpriority="high"`, preloaded, eager. Everything below the
  fold: `loading="lazy"`.
- Code-split routes. Use a null or invisible Suspense fallback rather than a spinner:
  a flash of skeleton on a fast chunk load is itself jank.
- Ship real HTML per shareable route (title, description, OG tags) even for an SPA;
  crawlers and link unfurlers do not run JS.

## The QA gate

Before calling a surface done, sweep every route at four viewports (390 and 768 in
WebKit, 1440 and 1920 in Chromium) and fail the build on any of:

- Horizontal overflow (any `scrollWidth > innerWidth`)
- Console errors
- Tap targets under 44px
- Type under 12px
- Images without alt text, broken heading hierarchy
- An ambient video showing a native play control (see video.md for the refusal test)

Test phones on WebKit, not scaled-down Chromium. The defects that reach real users are
Safari defects.
