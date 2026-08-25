---
name: animations
description: Build and debug accessible, performant animations in React and Next.js using CSS, Motion, or GSAP. Use for reveals, gestures, layout animation, parallax, scroll-linked effects, timelines, and page transitions.
---

# Animation System

Choose the lightest tool that cleanly expresses the interaction. Preserve the project's existing animation library and conventions when they are already established.

## Scope

**In scope:** CSS transitions and keyframes, Motion components and hooks, GSAP timelines and ScrollTrigger, React client boundaries, animation accessibility, cleanup, and performance.

**Out of scope:** General component architecture, visual design, and styling conventions unrelated to animation.

## Choose the Tool

1. **CSS first** for isolated hover, focus, color, opacity, and simple transform transitions.
2. **Motion by default** for state-driven animation, gestures, enter/exit presence, layout transitions, viewport reveals, and ordinary scroll-linked effects.
3. **GSAP only when needed** for precise multi-step sequencing, pinned or scrubbed ScrollTrigger timelines, or imperative animation that Motion cannot express cleanly.

Do not combine Motion and GSAP on the same property of the same element. Competing animation engines produce unstable transforms and cleanup bugs.

## Motion in Next.js

For a React Server Component that only renders Motion components with serializable props, use Motion's server-compatible entry point:

```tsx
import * as motion from "motion/react-client";

const Hero = () => (
  <motion.section
    initial={{ opacity: 0, y: 16 }}
    animate={{ opacity: 1, y: 0 }}
    transition={{ duration: 0.45 }}
  />
);
```

If the project already exposes a client boundary such as `@/components/motion`, keep using it rather than introducing a second convention.

Motion hooks require a Client Component:

```tsx
"use client";

import { motion, useReducedMotion } from "motion/react";

const AnimatedPanel = () => {
  const shouldReduceMotion = useReducedMotion();

  return (
    <motion.div
      initial={{ opacity: 0, y: shouldReduceMotion ? 0 : 16 }}
      animate={{ opacity: 1, y: 0 }}
    />
  );
};
```

- Prefer `whileInView` for a simple reveal; use `useInView` when visibility drives other logic.
- Use `useScroll` and `useTransform` for ordinary parallax or progress-driven effects.
- Use `AnimatePresence` only for elements that actually mount and unmount, with stable keys.
- Animate transforms and opacity when possible. Avoid measuring layout or animating width/height unless the visual requirement needs it.

## GSAP in React

Register plugins once at module scope. Create animations inside `useGSAP` with a scope so animations and ScrollTriggers are reverted on cleanup.

```tsx
"use client";

import { useRef } from "react";
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { useGSAP } from "@gsap/react";

gsap.registerPlugin(ScrollTrigger, useGSAP);

const ScrollSection = () => {
  const sectionRef = useRef<HTMLElement>(null);

  useGSAP(
    () => {
      gsap.from("[data-reveal]", {
        opacity: 0,
        stagger: 0.08,
        y: 24,
      });
    },
    { scope: sectionRef },
  );

  return <section ref={sectionRef}>{/* ... */}</section>;
};
```

- Prefer refs for single elements and scoped `data-*` selectors for groups.
- Use `contextSafe` for callbacks that create GSAP animations after the main `useGSAP` callback runs.
- Use `ease: "none"` for scrubbed animations where scroll position directly controls progress.
- Do not create timelines during render or register plugins inside a component.

## Accessibility and Performance

- Respect `prefers-reduced-motion`. Replace spatial movement, parallax, and autoplay with opacity-only or no animation when reduction is requested.
- Never make essential information depend on an animation completing.
- Preserve keyboard focus and DOM reading order during page or layout transitions.
- Avoid continuous animation when the page or element is not visible.
- Keep animation work off layout-sensitive properties when possible. Measure before animating only when necessary.
- Apply `will-change` sparingly and shortly before animation; persistent broad `will-change` usage wastes memory.
- Test mount/unmount behavior, route changes, resize, reduced motion, keyboard interaction, and a production build.

## Completion Checklist

- The lightest suitable tool is used.
- Server and Client Component boundaries remain intentional.
- Reduced-motion behavior is implemented.
- Effects, timelines, observers, and ScrollTriggers clean up correctly.
- Motion and GSAP do not compete for the same properties.
- The animation works after navigation, resize, and remount.
