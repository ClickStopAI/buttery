# Scroll architecture

The scroll doctrine in one line: desktop may be art-directed, touch is sacred.

## The two-mode shell

The shipped pattern renders every page inside an inset rounded frame, and the frame
behaves differently per device class. Breakpoint: `(max-width: 767.98px)`, watched with
`matchMedia().addEventListener("change")`, never a resize listener.

### Desktop (pointer + wheel): scroll inside the frame

```html
<div class="h-screen bg-white p-5">                <!-- white gutter -->
  <div class="frame relative h-full overflow-hidden rounded-[36px]">
    <div class="scroller absolute inset-0 overflow-y-auto overflow-x-hidden no-scrollbar">
      <!-- page content -->
    </div>
    <!-- nav and overlays absolutely positioned inside the frame -->
  </div>
</div>
```

```css
html, body { overflow: hidden; }     /* the document never scrolls on desktop */
.no-scrollbar { scrollbar-width: none; -ms-overflow-style: none; }
.no-scrollbar::-webkit-scrollbar { display: none; }
```

Smooth-wheel feel comes from Lenis, constructed on the inner scroller, desktop only:

```js
if (window.matchMedia("(prefers-reduced-motion: reduce)").matches) return;
const lenis = new Lenis({ wrapper: el, content: el.firstElementChild, autoRaf: true });
// cleanup: lenis.destroy()
```

Use Lenis defaults. They are: `lerp 0.1` (exponential smoothing, frame-rate
normalized), `smoothWheel true`, `syncTouch false` (touch stays native even if Lenis
somehow runs), `wheelMultiplier 1`, `overscroll true`. Do not pass a duration/easing
pair; lerp mode feels better and recovers gracefully when the user changes direction.

### Mobile (< 768px): the document scrolls

An inner scroll container on iOS kills pull-to-refresh, the rubber-band, and the
address-bar collapse. So on phones the frame becomes an in-flow rounded card and the
document itself scrolls:

```js
document.documentElement.classList.add("native-scroll");
scrollerRef = document.scrollingElement;
```

```css
html.native-scroll, html.native-scroll body { overflow: visible; }
```

No Lenis is constructed on mobile at all. Zero JS touches touch scrolling.

### Overlays that work in both modes

Keep a reference (context, store, or module variable) to "the active scroller": the
inner div on desktop, `document.scrollingElement` on mobile. Overlays like back-to-top
read scroll position from it. One capture-phase passive listener on `document` hears
scroll events from either mode:

```js
document.addEventListener("scroll", onScroll, { capture: true, passive: true });
```

## Unified anchor scrolling

One helper does every in-page jump so behavior never forks per page:

```js
function anchorScroll(target, offset = 20, smooth = true) {
  const behavior = smooth ? "smooth" : "auto";
  if (isMobile()) {
    const y = target.getBoundingClientRect().top + window.scrollY - offset;
    window.scrollTo({ top: y, behavior });
  } else {
    const s = scroller();  // the inner div
    const y = target.getBoundingClientRect().top - s.getBoundingClientRect().top + s.scrollTop - offset;
    s.scrollTo({ top: y, behavior });
  }
}
```

Pass `smooth = false` under reduced motion. Route changes reset scroll to top; a
hash in the URL scrolls into view after mount. Give sections `scroll-margin-top` so
anchored headings never hide under fixed chrome.

## Nav that gets out of the way

The mobile nav bar retracts on scroll-down and returns on scroll-up:

- Hide when `y > lastY + 6`, show when `y < lastY - 6` (the 6px deadband prevents
  jitter), always show near the top (`y < 80`).
- `transform: translateY(-150%)` / `translateY(0)`, 280ms MATERIAL curve
  `cubic-bezier(0.4, 0, 0.2, 1)`.
- Listener is passive; the handler is two integer compares. No throttling needed
  because the work is cheaper than the throttle.
- Under reduced motion the transition is removed so the bar snaps.

## Menu overlays over video

Full-screen `backdrop-filter` blur over a playing video stalls phone GPUs. Mobile menu
sheets are opaque instead, contained with `overscroll-behavior: contain`, and lock the
root scroll while open. Desktop megamenus animate opacity and 8px of translateY over
200ms, and carry an invisible hover bridge spanning the gap between trigger bar and
panel so hover survives the crossing:

```html
<div class="absolute -top-4 left-0 right-0 h-4"></div>
```

## Measurement rules

- `matchMedia` change events over `resize` listeners, always.
- `ResizeObserver` for element measurement.
- Any hand-rolled physics (drag carousels, coverflows) runs in a single
  requestAnimationFrame loop that is cancelled when idle, writes transforms directly to
  `element.style` (never through framework state per frame), and settles exponentially:
  `pos += (target - pos) * 0.14` per frame, stopping below a 0.0005 remainder.
