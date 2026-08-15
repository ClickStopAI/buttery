---
name: buttery
description: >
  Make any web interface feel buttery smooth. Use this skill whenever you are building or
  polishing anything a human scrolls, hovers, taps, or watches: landing pages, marketing
  sites, dashboards, web apps, hero sections, carousels, accordions, background video,
  page transitions, or micro-interactions. Also use it when the user says things like
  "make it feel smooth", "it feels janky", "add polish", "make it feel premium", "the
  scroll feels off", or "the animations feel cheap", even if they never say the word
  "buttery". It is a set of portable, framework-agnostic rules with exact numbers:
  easing curves, durations, stagger timing, scroll architecture, 60fps video practice,
  and reduced-motion fallbacks, all extracted from a shipped production site.
---

# Buttery

Butter for ClickStop. A portable doctrine for making interfaces feel smooth, calm, and
expensive. Every number in this file shipped on [clickstop.ai](https://clickstop.ai) and
survived real QA on real phones. Nothing here is theoretical.

Works with any stack: plain CSS, Tailwind, React, Vue, Svelte, or hand-rolled JS.
The rules are written against the web platform, not a framework.

## The one-sentence doctrine

Animate only what the compositor can carry, ease everything out exponentially, stagger
entrances in ~90ms steps, never hijack touch scrolling, ship video at 60fps behind a
poster that can never fail, and give reduced-motion users a designed alternative rather
than a broken page.

## The easing vocabulary (memorize these five)

Pick from this table. Do not invent new curves per element; a site feels coherent when
every motion shares a small family of easings.

| Name | cubic-bezier | Use for |
|---|---|---|
| EXPO_OUT | `cubic-bezier(0.16, 1, 0.3, 1)` | Reveals, entrances, panel swaps. The default. |
| QUINT_OUT | `cubic-bezier(0.23, 1, 0.32, 1)` | Shape morphs, accordions, indicator dots |
| BACK_OUT | `cubic-bezier(0.34, 1.56, 0.64, 1)` | Arrows, success pops. Overshoot; use sparingly. |
| EXPO_SOFT | `cubic-bezier(0.19, 1, 0.22, 1)` | Large fills and sweeps inside a control |
| MATERIAL | `cubic-bezier(0.4, 0, 0.2, 1)` | UI chrome hiding and showing (nav retract) |

Why ease-out everywhere: motion that starts fast and lands softly reads as the interface
responding instantly and then settling. Ease-in starts sluggish; linear reads mechanical.

## Durations

- Micro-interactions (hover, focus, arrows): 200 to 400ms.
- Entrances and reveals: 400 to 600ms.
- Shape morphs and choreographed controls: 600 to 800ms.
- Anything over 800ms must be ambient (marquees, slow washes), never blocking.

## Reveal-on-scroll (the whole system is 5 rules)

1. Animate `opacity` 0 to 1 and `transform: translateY(24px)` to 0. Nothing else.
2. Duration 400ms, EXPO_OUT.
3. Fire with an IntersectionObserver whose rootMargin is `-80px`: the animation starts
   just before the element reaches the viewport edge, so it is already finishing as the
   eye arrives. The user should catch motion mid-flight, never wait for it.
4. One-shot. Once revealed, an element never animates again. Re-triggering reveals on
   scroll-up makes a page feel nervous; one-shot makes it feel calm.
5. Stagger siblings by index: `delay = min(index * 90ms, 360ms)`. The cap matters: in a
   long list, item 12 must not arrive a second late. For dense grids, stagger by
   `index % columns` so each row restarts the ladder.

Under reduced motion, render the plain element. Do not mount the animation at all.

## Hero entrance choreography

A hero earns the "wow" in the first 1.2 seconds. The pattern:

1. Split the H1 into words. Each word: `opacity 0, translateY(40px)` to rest, 600ms
   EXPO_OUT, 80ms apart.
2. If one word carries an accent (gradient or color), give it its own beat roughly 150ms
   after the last plain word lands.
3. Subtitle: blur-in. `opacity 0, blur(10px), translateY(20px)` to rest, 600ms ease-out,
   delayed ~400ms.
4. CTA row: same blur-in, delayed ~600ms.

Everything overlaps; nothing waits for the previous step to finish. Total choreography
stays under 1.2s so repeat visitors are never held hostage.

## Scroll

- Never hijack scrolling on touch devices. Phones keep 100% native physics, including
  pull-to-refresh and rubber-banding. Smooth-scroll libraries (such as Lenis) run on
  desktop pointer-wheel input only, and are never constructed when the user prefers
  reduced motion.
- If you use Lenis, its defaults are correct: `lerp 0.1`, `smoothWheel true`,
  `syncTouch false`. Resist tuning.
- Scroll listeners are always `{ passive: true }`. Prefer `matchMedia` change events
  over `resize` listeners, and `ResizeObserver` over window measurements.
- Anchor scrolling goes through one shared helper so every in-page jump behaves the
  same, and switches to instant (`behavior: "auto"`) under reduced motion.

The full two-mode shell architecture (desktop scrolls inside an inset rounded frame,
phones scroll the document) is in [references/scroll.md](references/scroll.md).

## Micro-interactions

- Card hover: `translateY(-3px)` plus a soft shadow, 250 to 500ms. Tint the shadow with
  your brand's dark color, never pure black: `0 12px 32px rgba(<brand-ink>, 0.10)`.
  Black shadows read as dirt; brand-tinted shadows read as depth.
- Buttons may morph (radius, fill sweep, arrows trading places) with layered timings:
  shell 600ms QUINT_OUT, internals 800ms with BACK_OUT arrows. Press state:
  `scale(0.95)`.
- Accordions animate `grid-template-rows: 0fr` to `1fr` on a grid wrapper with an
  `overflow: hidden` child. No JS height measuring, works with any content, 400ms
  QUINT_OUT.
- Marquees duplicate their row for a seamless loop, run 26 to 70s linear, fade edges
  with a CSS mask, and pause on hover.

Exact recipes with code: [references/motion.md](references/motion.md).

## Video (the 60fps rule)

Two rules produce the "how is this so smooth" reaction:

1. **Ship 60fps files.** If your source is 24 or 30fps, interpolate:
   `ffmpeg -i in.mp4 -vf "minterpolate=fps=60:mi_mode=mci:mc_mode=aobmc:vsme=1" -c:v libx264 -crf 18 -pix_fmt yuv420p out60.mp4`
   Then encode web variants at crf ~23. A 720p 60fps loop can land near 1MB.
2. **A visitor never sees a play button.** The poster image always renders and is the
   LCP element. The video is invisible until it fires a real `playing` event, fades in
   over 600ms, and if autoplay is refused or nothing plays within a 6s grace window, the
   video element is removed entirely and the poster is the final state.

Attributes that make mobile autoplay possible at all:
`autoplay muted loop playsinline preload="metadata" poster="..."`.
Pause off-screen loops with an IntersectionObserver at threshold 0.2.

Full architecture, seamless-loop technique, and the WebKit test that proves the
fallback: [references/video.md](references/video.md).

## Performance floor

- Animate `transform`, `opacity`, and `filter`. Treat anything else as a deliberate
  exception you can defend (accordion rows, a button radius).
- `will-change` only where measurement shows it helps (hero word spans, 3D carousel
  cards). Spraying it costs memory and can hurt.
- Explicit dimensions or aspect-ratio on every image and video slot. Zero layout shift
  from media is non-negotiable.
- Self-host fonts as woff2 with `font-display: swap`. Declare variable fonts with
  `font-weight: 100 900` so the browser never synthesizes faux bold, which changes
  glyph widths and reflows headings.
- Size type and spacing with `clamp()` so nothing jumps at breakpoints.
- Lazy-load below-fold images and code-split routes with a null loading fallback, not a
  spinner.

Details and the QA gate checklist: [references/performance.md](references/performance.md).

## Reduced motion is a design target, not a kill switch

Every animated surface degrades to a designed alternative. The five strategies, from
cheapest to most involved:

1. CSS `@media (prefers-reduced-motion: reduce)` overrides.
2. Motion classes applied through a motion-safe variant so they never attach.
3. Components that early-return plain markup instead of mounting animation nodes.
4. Component substitution: a carousel becomes a static grid.
5. Behavioral gating: smooth-scroll never constructed, ambient video never mounted,
   entrance props skipped, scroll behavior switched to instant.

A frozen marquee should become a wrapped grid. A paused video is a permanent play
button, so remove the element instead. Worked examples:
[references/reduced-motion.md](references/reduced-motion.md).

## The cheat sheet

```
EXPO_OUT      cubic-bezier(0.16, 1, 0.3, 1)      reveals, entrances, swaps
QUINT_OUT     cubic-bezier(0.23, 1, 0.32, 1)     morphs, accordions, dots
BACK_OUT      cubic-bezier(0.34, 1.56, 0.64, 1)  arrows, pops (overshoot)
EXPO_SOFT     cubic-bezier(0.19, 1, 0.22, 1)     fills, sweeps
MATERIAL      cubic-bezier(0.4, 0, 0.2, 1)       chrome hide/show

Reveal        400ms | y 24px -> 0 | one-shot | IO rootMargin -80px
Stagger       90ms/item, cap 360ms | grids: index % columns
Hero words    600ms | 80ms apart | y 40px | accent word +150ms
Hero blur-in  600ms | blur(10px) -> 0 | delays 400/600ms
Card hover    250-500ms | translateY(-3px) | brand-ink shadow at 10%
Button morph  shell 600ms | internals 800ms | press scale 0.95
Accordion     400ms | grid-rows 0fr -> 1fr | icon rotate 45deg 300ms
Carousel      700ms EXPO_OUT slide | 5s auto-advance, only while visible
Marquee       26-70s linear | duplicated row | edge mask | pause on hover
Video         60fps h264 | poster-first | 600ms fade | 6s grace | IO 0.2
Scroll        native on touch, always | Lenis defaults desktop only
Listeners     passive: true | matchMedia > resize | ResizeObserver
```

## How to apply this skill

1. Audit what exists: list every animated surface and every scroll behavior.
2. Replace ad-hoc easings and durations with the vocabulary above.
3. Wire the reveal system and stagger ladder onto section content.
4. Fix video to the poster-first architecture; re-render files at 60fps.
5. Add the reduced-motion path for every surface you touched.
6. Verify on a real phone or WebKit emulation, not just Chrome desktop: check that
   nothing hijacks touch scroll, no play button ever appears, and nothing shifts layout.
