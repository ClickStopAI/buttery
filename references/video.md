# Video: 60fps files, poster-first playback

Background video is where "buttery" is won or lost. Two halves: the files themselves,
and the playback architecture around them.

## Half one: the files

### 60fps is mandatory

24 and 30fps loops read as "web video". 60fps reads as part of the interface. If the
source is lower framerate (stock, AI-generated, screen capture), interpolate up:

```bash
ffmpeg -i in.mp4 \
  -vf "minterpolate=fps=60:mi_mode=mci:mc_mode=aobmc:vsme=1" \
  -c:v libx264 -crf 18 -pix_fmt yuv420p out60.mp4
```

Motion-compensated interpolation (`mci` + `aobmc`) is slow to encode and worth it.
QA the result frame-by-frame around fast motion; interpolation artifacts cluster there.

### Encode ladder

Master at crf 18, then encode web variants at crf ~23. Real shipped numbers to aim at:

| Use | Resolution | fps | Bitrate | Size (36s loop) |
|---|---|---|---|---|
| Desktop hero | 1920x1080 | 60 | ~1.4 Mbps | ~6 MB |
| Mobile hero | 1280x720 | 60 | ~240 kbps | ~1.1 MB |
| Section loop | 1080x1080 | 60 | ~0.9-1.2 Mbps | 2-4 MB |

h264 High, `yuv420p` (required for Safari), `-movflags +faststart` for streaming.
Posters are 20-40 KB JPEGs extracted from frame one:

```bash
ffmpeg -i loop.mp4 -frames:v 1 -vf scale=1280:-2 poster.jpg
```

### Seamless loops

- Prompt or design the motion to return to its opening frame.
- Verify the seam by extracting first and last frames and diffing visually.
- If the seam shows, cut the clip as a palindrome (forward then reversed) at slightly
  different speeds (0.55x / 0.7x) so the join is invisible.

## Half two: playback architecture

### The rule

A visitor never sees a play button. Either the loop is playing, or they see a still
frame, and nothing in between. Playback is an enhancement that must prove itself.

### The three-state machine

1. **Poster first.** An `<img>` with the poster always renders underneath and is the
   LCP element: `fetchpriority="high"`, `decoding="async"`, and preloaded from the HTML
   head (`<link rel="preload" as="image" href="poster.jpg" fetchpriority="high">`).
   Never let the video be your LCP; it resolves seconds later.
2. **Video proves itself.** The `<video>` renders invisible and only fades in when it
   fires a real `playing` event. Fade: 600ms. Use `filter: opacity(0|1)` rather than
   the `opacity` property if callers also set opacity classes for scrim compositing.
3. **Failure is silent and final.** If `play()` rejects, an `error` fires, or no
   `playing` event arrives within a 6000ms grace window, unmount the video element
   entirely. The poster is the designed final state, not a degraded one.

Start the grace timer when the video first becomes visible in the viewport, not on
mount, so below-fold loops are not failed before they were ever asked to play.

### The element

```html
<video autoplay muted loop playsinline
       preload="metadata" poster="poster.jpg" src="loop.mp4"
       disablepictureinpicture></video>
```

- `muted` + `playsinline` are what make mobile autoplay legal at all.
- `preload="metadata"`, never `auto`.
- `pointer-events: none` on ambient loops.
- Kill WebKit's injected controls, which can paint a giant play glyph over a paused
  ambient video:

```css
.loop-video::-webkit-media-controls,
.loop-video::-webkit-media-controls-start-playback-button,
.loop-video::-webkit-media-controls-play-button,
.loop-video::-webkit-media-controls-panel {
  display: none !important;
  -webkit-appearance: none; appearance: none;
}
```

### Responsive sources

Resolve the source once, before first paint, and never swap it mid-flight (Safari
restarts loading on a source swap and can strand playback):

```js
const source = matchMedia("(max-width: 820px)").matches ? mobileSrc : desktopSrc;
```

### Off-screen discipline

IntersectionObserver at threshold 0.2: `video.play()` when intersecting, `video.pause()`
when not. Phones will not decode six loops at once, and they should not have to.

### Reduced motion

Do not mount the video element at all. A paused video is a permanent play button; the
poster is the reduced-motion experience.

## Prove it with a test

The failure mode you must test is autoplay refusal, and you cannot test it by hoping.
In Playwright WebKit (not Chromium; the defects that reach users are Safari defects),
monkey-patch play to simulate refusal before the page loads:

```js
await context.addInitScript(() => {
  HTMLMediaElement.prototype.play = function () {
    this.dispatchEvent(new Event("error"));
    return Promise.reject(new DOMException("denied", "NotAllowedError"));
  };
});
```

Then scroll the page in steps and assert: zero visible video elements, a visible
poster, and no native play control. Run the same sweep with autoplay allowed and assert
the opposite. Both directions, every release.
