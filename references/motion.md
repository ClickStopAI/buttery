# Motion recipes

Exact implementations for the animation patterns in the Buttery doctrine. Examples use
plain CSS and small JS; React/Motion variants noted where the shipped site used them.

## The reveal component

The entire reveal-on-scroll system is about 30 lines. React + Motion version as shipped:

```tsx
const EXPO_OUT = [0.16, 1, 0.3, 1] as const;

function Reveal({ children, delay = 0, y = 24, className }) {
  const reduced = useReducedMotion();
  if (reduced) return <div className={className}>{children}</div>;
  return (
    <motion.div
      className={className}
      initial={{ opacity: 0, y }}
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true, margin: "-80px" }}
      transition={{ duration: 0.4, delay, ease: EXPO_OUT }}
    >
      {children}
    </motion.div>
  );
}
```

Framework-free version:

```html
<style>
  .reveal { opacity: 0; transform: translateY(24px); }
  .reveal.in {
    opacity: 1; transform: translateY(0);
    transition: opacity .4s cubic-bezier(.16,1,.3,1), transform .4s cubic-bezier(.16,1,.3,1);
    transition-delay: var(--reveal-delay, 0s);
  }
  @media (prefers-reduced-motion: reduce) {
    .reveal { opacity: 1; transform: none; transition: none; }
  }
</style>
<script>
  const io = new IntersectionObserver((entries) => {
    for (const e of entries) if (e.isIntersecting) {
      e.target.classList.add("in");
      io.unobserve(e.target);            // one-shot
    }
  }, { rootMargin: "-80px" });
  document.querySelectorAll(".reveal").forEach((el) => io.observe(el));
</script>
```

## The stagger ladder

Assign delays by index with a hard cap so long lists never straggle:

```js
delay = Math.min(i * 0.09, 0.36)   // house default: 90ms step, 360ms cap
```

Variants that shipped, by context:

| Step | Cap | When |
|---|---|---|
| 90ms | 360ms | Default for card grids, steps, FAQ items |
| 70ms | 350ms | Denser grids (6-up) |
| 60ms | none | Short lists of 3 or fewer |
| 50ms | none | Long process rows where items are small |
| 80-120ms | n/a | Second element of a heading-then-body pair |

For a large grid, reset the ladder per row: `delay = (i % columns) * 0.05`. A 16-item
grid must never accumulate a 1.2s tail.

## Hero choreography

Word-by-word H1 (split on spaces, each word an inline-block span):

```
word i:  opacity 0, translateY(40px) -> rest | 600ms EXPO_OUT | delay 80ms * i
accent:  its own span, delay = last word + ~150ms
```

Give the word spans `will-change: transform` (this is one of only two sanctioned
`will-change` sites). Blur-in for subtitle and CTAs:

```
from: opacity 0, filter blur(10px), translateY(20px)
to:   opacity 1, blur 0, y 0 | 600ms ease-out | subtitle delay .4s, CTAs .6s
```

Reduced motion: return the raw string and plain divs.

## Card hover

```css
.card {
  transition: all .25s ease;               /* up to .5s for larger cards */
}
.card:hover {
  transform: translateY(-3px);
  box-shadow: 0 12px 32px rgba(21, 67, 89, 0.10);  /* swap in YOUR ink color */
}
```

The shadow is always the brand's darkest color at ~10% alpha, never neutral black.
Optional brand wash on hover: a radial gradient overlay at 8% alpha fading in over
500ms (`transition: opacity .5s`).

Arrow reveal inside a card:

```css
.card .arrow {
  opacity: 0; transform: translateX(-6px);
  transition: all .3s cubic-bezier(.34,1.56,.64,1);   /* BACK_OUT */
}
.card:hover .arrow { opacity: 1; transform: translateX(0); }
```

## The morphing button

Layered timings on one control. Shell and internals run on different clocks:

| Layer | Duration | Easing | Motion |
|---|---|---|---|
| Shell | 600ms | QUINT_OUT | border-radius 100px to 12px, border fades |
| Entering arrow | 800ms | BACK_OUT | from left: -25% to a 16px inset |
| Exiting arrow | 800ms | BACK_OUT | from 16px inset to right: -25% |
| Label | 800ms | ease-out | slides 12px toward the exiting arrow |
| Fill circle | 800ms | EXPO_SOFT | width 16px to 240% (aspect-square), opacity 0 to 1 |
| Press | instant | n/a | `active: scale(0.95)` |

Size the fill circle in percent of the button's own width with `aspect-ratio: 1`, not
fixed pixels, so it always outgrows wide labels.

## Accordion without height hacks

```html
<div class="acc" data-open="false">
  <div class="acc-inner"><!-- content --></div>
</div>
<style>
  .acc {
    display: grid; grid-template-rows: 0fr;
    transition: grid-template-rows .4s cubic-bezier(.23,1,.32,1);
  }
  .acc[data-open="true"] { grid-template-rows: 1fr; }
  .acc-inner { overflow: hidden; }
</style>
```

No measuring, works with dynamic content, buttery at any height. The +/x icon rotates
45deg over 300ms with the same QUINT_OUT curve.

## Marquee

- Duplicate the row once; animate the track `translateX(0)` to `translateX(-50%)`,
  linear, infinite. 26s for a display-type divider; 45 to 70s for content rows, with
  adjacent rows at different speeds and directions.
- Edge fade: `mask-image: linear-gradient(to right, transparent, black 8%, black 92%, transparent)`.
- Pause on hover: `animation-play-state: paused` on the track.
- Mark the duplicate row `aria-hidden="true"`.

## Carousel

- Slide: `translateX(-index * 100%)`, 700ms EXPO_OUT.
- Auto-advance every 5s, but only while on screen: gate the interval with an
  IntersectionObserver at threshold 0.3.
- Pause on hover and on pointer down. Swipe commits at 40px of travel.
- `touch-action: pan-y` so vertical page scrolling is never stolen.
- Indicator dots stretch 6px to 26px over 450ms QUINT_OUT.

## Ambient touches

- Scroll cue: 6px vertical drift, 1.6s ease-in-out infinite, killed under reduced motion.
- Film grain: an SVG feTurbulence overlay at ~6% opacity animated with
  `steps(5)` over 1.2s so it judders like grain instead of sliding. Give each SVG
  filter instance a unique id; some engines reuse the first one they see.
- Hero scrims over video are multi-stop gradients (light at top, deep at the base),
  never a single flat overlay.
