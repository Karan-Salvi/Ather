# Custom Cursor States — Design Spec

**Date:** 2026-07-05  
**Status:** Approved  

---

## Overview

Build a greenfield section-aware custom cursor system for the Aether AI landing page (`index.html`). As the cursor moves across the page it morphs between distinct states depending on the hovered context. No custom cursor exists in the codebase today — this is a full build.

**Approach:** Pure CSS transitions for all size/shape/opacity morphing. A vanilla `requestAnimationFrame` lerp loop for cursor following. No GSAP dependency for the cursor system. State classes on `#custom-cursor` drive all visual changes.

---

## Cursor States

| State class | Trigger zone | Size | Appearance | Label/icon |
|---|---|---|---|---|
| *(none)* | Default / empty space | 12px | Solid white dot | none |
| `state-view` | Case study cards (`#horizontal-scroll > div`) | 80px | Solid white circle | "View" (black text) |
| `state-play` | Watch Demo button (`#hero-btn-2`) | 80px | Solid white circle | Play SVG icon |
| `state-explore` | Dashboard mockup (`.dashboard-shell`) | 80px | Solid white circle | "Explore" (black text) |
| `state-select` | Pricing cards (structural: `.max-w-6xl > div`, scoped to the pricing grid which has `max-w-6xl mx-auto`) | 56px | Solid white circle | "Select" (black text) |
| `state-node` | Workflow timeline nodes (`.step-node`) | 36px | Transparent, white border ring | none |
| `state-drag` | Logo marquee (`.marquee-container`) | 64px | Solid white circle | Drag/arrow SVG icon |
| `state-hidden` | Nav links, footer links (`nav a, footer a`) | — | opacity: 0 | none |

---

## HTML

One element, placed immediately before `</body>`:

```html
<div id="custom-cursor" aria-hidden="true">
  <span id="cursor-label"></span>
  <svg id="cursor-icon-play" class="cursor-icon" viewBox="0 0 24 24">
    <polygon points="7,4 20,12 7,20" fill="white"/>
  </svg>
  <svg id="cursor-icon-drag" class="cursor-icon" viewBox="0 0 24 24">
    <path d="M4 12h16M8 8l-4 4 4 4M16 8l4 4-4 4" stroke="white" stroke-width="1.5" fill="none" stroke-linecap="round" stroke-linejoin="round"/>
  </svg>
</div>
```

---

## CSS

Added inside the existing `<style>` block. Entire system scoped to `@media (hover: hover) and (pointer: fine)` — touch devices and stylus inputs retain native cursor behavior automatically.

```css
@media (hover: hover) and (pointer: fine) {
  * { cursor: none !important; }

  #custom-cursor {
    position: fixed;
    top: 0; left: 0;
    width: 12px; height: 12px;
    background: rgba(255,255,255,0.9);
    border-radius: 9999px;
    border: 1.5px solid transparent;
    pointer-events: none;
    z-index: 99999;
    display: flex; align-items: center; justify-content: center;
    transform: translate(-50%, -50%);
    transition:
      width 0.35s cubic-bezier(0.23,1,0.32,1),
      height 0.35s cubic-bezier(0.23,1,0.32,1),
      background 0.35s ease,
      border-color 0.35s ease,
      opacity 0.2s ease;
    will-change: transform;
  }

  #cursor-label, .cursor-icon {
    opacity: 0;
    font-size: 11px;
    font-weight: 600;
    letter-spacing: 0.08em;
    color: #000;
    font-family: 'JetBrains Mono', monospace;
    pointer-events: none;
    transition: opacity 0.15s ease;
    user-select: none;
  }
  .cursor-icon {
    width: 16px; height: 16px;
    position: absolute;
  }

  #custom-cursor.state-view,
  #custom-cursor.state-explore {
    width: 80px; height: 80px;
  }
  #custom-cursor.state-select {
    width: 56px; height: 56px;
  }
  #custom-cursor.state-view #cursor-label,
  #custom-cursor.state-explore #cursor-label,
  #custom-cursor.state-select #cursor-label {
    opacity: 1;
  }

  #custom-cursor.state-play {
    width: 80px; height: 80px;
  }
  #custom-cursor.state-play #cursor-icon-play { opacity: 1; }

  #custom-cursor.state-drag {
    width: 64px; height: 64px;
  }
  #custom-cursor.state-drag #cursor-icon-drag { opacity: 1; }

  #custom-cursor.state-node {
    width: 36px; height: 36px;
    background: transparent;
    border-color: rgba(255,255,255,0.6);
  }

  #custom-cursor.state-hidden { opacity: 0; }
}

@media (prefers-reduced-motion: reduce) {
  #custom-cursor {
    transition: opacity 0.1s ease;
  }
  #cursor-label, .cursor-icon {
    transition: none;
  }
}
```

---

## JavaScript

Single `initCustomCursor()` function, called at the bottom of the existing `DOMContentLoaded` block. Three responsibilities:

### 1. Follow loop

`requestAnimationFrame` lerp at factor `0.12` (~8 frames to reach target). Starts offscreen at `(-200, -200)` to avoid a flash on page load.

```js
function initCustomCursor() {
  const cursor = document.getElementById('custom-cursor');
  const label = document.getElementById('cursor-label');
  if (!cursor) return;
  if (!window.matchMedia('(hover: hover) and (pointer: fine)').matches) return;

  let mx = -200, my = -200;
  let cx = -200, cy = -200;

  document.addEventListener('mousemove', e => { mx = e.clientX; my = e.clientY; });

  function loop() {
    cx += (mx - cx) * 0.12;
    cy += (my - cy) * 0.12;
    cursor.style.transform = `translate(${cx}px, ${cy}px) translate(-50%, -50%)`;
    requestAnimationFrame(loop);
  }
  loop();
```

### 2. State machine

```js
  let currentState = '';
  let labelTimeout;

  function setCursorState(state, labelText = '') {
    if (state === currentState) return;
    currentState = state;

    label.style.opacity = '0';
    clearTimeout(labelTimeout);
    labelTimeout = setTimeout(() => {
      label.textContent = labelText;
    }, 150);

    cursor.className = state ? `state-${state}` : '';
  }

  function clearState() {
    setCursorState('');
  }
```

The 150ms label fade-out window covers the case where the cursor moves directly from one labelled zone into another without crossing neutral space — the text swaps during the invisible window rather than flashing.

### 3. Zone wiring

```js
  const zones = [
    { selector: '#horizontal-scroll > div',                        state: 'view',    label: 'View'    },
    { selector: '#hero-btn-2',                                     state: 'play',    label: ''        },
    { selector: '.dashboard-shell',                                state: 'explore', label: 'Explore' },
    { selector: '.max-w-6xl.mx-auto > div',                        state: 'select',  label: 'Select'  },
    { selector: '.step-node',                                      state: 'node',    label: ''        },
    { selector: '.marquee-container',                              state: 'drag',    label: ''        },
    { selector: 'nav a, footer a',                                 state: 'hidden',  label: ''        },
  ];

  zones.forEach(({ selector, state, label: lbl }) => {
    document.querySelectorAll(selector).forEach(el => {
      el.addEventListener('mouseenter', () => setCursorState(state, lbl));
      el.addEventListener('mouseleave', clearState);
    });
  });
}

initCustomCursor();
```

---

## Known Fragilities

- **Pricing card selector** uses `.max-w-6xl.mx-auto > div` to distinguish the pricing grid from the testimonials grid (which also uses `md:grid-cols-3` but lacks `max-w-6xl`). Both grids are direct `> div` parents, so the `max-w-6xl` scoping is what makes this safe. Adding a `pricing-card` class to each card at implementation time would make this robust and remove the structural dependency.
- **`state-node` on mobile:** `.step-node` elements are hidden on mobile via CSS (`display: none !important`), so `mouseenter` can never fire on them on touch viewports. The JS `(hover: hover)` guard covers this anyway, but worth noting the desktop-only nature of this state.

---

## Constraints Honoured

- No new colors — all states use white/black/gray only
- Touch devices: entire system disabled via `@media (hover: hover) and (pointer: fine)` in both CSS and JS
- `prefers-reduced-motion`: size transitions instant, opacity transitions retained at 0.1s
- No GSAP used for cursor — zero risk of ScrollTrigger `overwrite` conflicts
- `aria-hidden="true"` on cursor element — invisible to screen readers
