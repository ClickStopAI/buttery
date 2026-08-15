# Reduced motion: five strategies

`prefers-reduced-motion` users get a designed alternative, never a broken or hollowed
page. Choose the cheapest strategy that produces a genuinely good static experience.

## 1. CSS media query overrides

For pure-CSS animation, override in place:

```css
@media (prefers-reduced-motion: reduce) {
  .animate-cue { animation: none; }
}
```

But do not stop at `animation: none` if the frozen state is wrong. A marquee frozen
mid-scroll clips half a card. Rebuild it as a real layout:

```css
@media (prefers-reduced-motion: reduce) {
  .marquee-track { animation: none !important; flex-wrap: wrap; row-gap: 1.25rem; justify-content: center; }
  .marquee-mask { mask-image: none; }
  .marquee-track[aria-hidden="true"] { display: none; }  /* the seamless-loop duplicate */
}
```

The frozen marquee becomes a wrapped, centered grid.

## 2. Motion-safe class application

Apply animation classes through a motion-safe variant so they never attach in the
first place (Tailwind: `motion-safe:animate-[...]`; plain CSS: put the animation
inside `@media (prefers-reduced-motion: no-preference)`). Cleaner than overriding.

## 3. Early-return plain markup

Animated components check the preference and return plain DOM, mounting no animation
machinery at all:

```tsx
const reduced = useReducedMotion();
if (reduced) return <div className={className}>{children}</div>;
return <motion.div ...>...</motion.div>;
```

Framework-free equivalent: gate the IntersectionObserver setup on
`matchMedia("(prefers-reduced-motion: reduce)").matches` and add the revealed state
class immediately.

## 4. Component substitution

When the animated thing is structural, swap the whole component:

```tsx
return reduced ? <StaffGrid members={team} /> : <StaffCarousel members={team} />;
```

A carousel becomes a static grid. This is the honest version of "reduced": same
content, calm presentation, nothing half-disabled.

## 5. Behavioral gating

The subtle one. Motion preference changes runtime behavior, not just styles:

- Smooth-scroll libraries are never constructed.
- Ambient `<video>` elements are never mounted. A paused video is a permanent play
  button; the poster is the experience.
- Entrance animations skip via `initial={reduced ? false : {...}}` (Motion) so the
  element appears settled without unmount/remount.
- UI chrome transitions set to none so bars snap instead of sliding.
- `scrollIntoView`/`scrollTo` behavior switches from `"smooth"` to `"auto"`.
- Post-scroll focus delays drop from ~700ms to 0.
- Physics components (drag carousels) jump to target and paint once instead of
  running their settle loop.

## The audit question

For every animated surface, ask: "If this never moved, what would a designer ship?"
Build that. If the answer is just the moving version paused at frame zero, you have
not designed the fallback yet.
