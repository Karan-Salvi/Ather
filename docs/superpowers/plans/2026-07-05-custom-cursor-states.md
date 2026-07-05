# Custom Cursor States Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a greenfield section-aware custom cursor system in `index.html` that morphs between 8 distinct visual states as the user hovers different page zones.

**Architecture:** One `#custom-cursor` element with child label/icon content, driven entirely by CSS modifier state classes (`state-view`, `state-play`, etc.) and a centralized `setCursorState()` JS function. A vanilla `requestAnimationFrame` lerp loop tracks mouse position. No GSAP involved — zero risk of ScrollTrigger conflicts.

**Tech Stack:** Vanilla HTML/CSS/JS, inline SVG icons, CSS `transition` for morphing, `requestAnimationFrame` for follow loop.

## Global Constraints

- No new colors — all cursor states must use white (`rgba(255,255,255,0.9)`), black (`#000`), or gray only
- Entire system scoped to `@media (hover: hover) and (pointer: fine)` in both CSS and JS — touch devices retain native cursors
- `prefers-reduced-motion`: size transitions become instant, opacity crossfades retained at 0.1s
- All new JS wrapped in `try { ... } catch (e) { console.error('Cursor states failed:', e) }` so a bug here cannot break the loader, hero entrance, or ScrollTrigger animations
- Do not touch the lerp follow loop (it doesn't exist yet — build it once and leave it alone)
- `aria-hidden="true"` on `#custom-cursor`
- Single `setCursorState(state, label)` function + `cursorZones` array — no one-off per-section listener blocks

---

## File Map

| File | Change |
|---|---|
| `index.html` | 3 targeted edits: (1) insert `#custom-cursor` HTML before `</body>`, (2) append cursor CSS inside `<style>`, (3) append `initCustomCursor()` call at bottom of `DOMContentLoaded` block |

---

### Task 1: Add `#custom-cursor` HTML element

**Files:**
- Modify: `index.html` — insert before the closing `</body>` tag (line ~3689)

**Interfaces:**
- Produces: `#custom-cursor`, `#cursor-label`, `#cursor-icon-play`, `#cursor-icon-drag` DOM elements consumed by Tasks 2 and 3

- [ ] **Step 1: Locate insertion point**

  Find the closing `</body>` tag in `index.html`. It is on line 3689 (the last line before `</html>`). The new element goes immediately before it, after the closing `</script>` tag at line 3688.

- [ ] **Step 2: Insert the cursor element**

  Add the following immediately before `</body>`:

  ```html
  <!-- ── Custom Cursor ─────────────────────────────────────────────────── -->
  <div id="custom-cursor" aria-hidden="true">
    <span id="cursor-label"></span>
    <svg id="cursor-icon-play" class="cursor-icon" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
      <polygon points="7,4 20,12 7,20" fill="white"/>
    </svg>
    <svg id="cursor-icon-drag" class="cursor-icon" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
      <path d="M4 12h16M8 8l-4 4 4 4M16 8l4 4-4 4" stroke="white" stroke-width="1.5" fill="none" stroke-linecap="round" stroke-linejoin="round"/>
    </svg>
  </div>
  ```

- [ ] **Step 3: Verify in browser**

  Open `index.html` in a browser. Open DevTools → Elements. Confirm `#custom-cursor` exists as a direct child of `<body>` with three children: `#cursor-label`, `#cursor-icon-play`, `#cursor-icon-drag`.

  The element will be invisible (no styles yet) — that's expected.

- [ ] **Step 4: Commit**

  ```bash
  git add index.html
  git commit -m "feat: add #custom-cursor HTML element with label and icon slots"
  ```

---

### Task 2: Add cursor CSS

**Files:**
- Modify: `index.html` — append inside the `<style>` block (currently ends at line ~685, just before `</style>`)

**Interfaces:**
- Consumes: `#custom-cursor`, `#cursor-label`, `.cursor-icon` from Task 1
- Produces: All visual state rules consumed by Task 3's JS state machine

- [ ] **Step 1: Locate the `</style>` closing tag**

  The inline `<style>` block closes at approximately line 685 (just before `</head>`). Insert the new CSS rules immediately before `</style>`.

- [ ] **Step 2: Insert the CSS**

  ```css
  /* ── Custom Cursor ───────────────────────────────────────────────────── */
  @media (hover: hover) and (pointer: fine) {
    * { cursor: none !important; }

    #custom-cursor {
      position: fixed;
      top: 0;
      left: 0;
      width: 12px;
      height: 12px;
      background: rgba(255, 255, 255, 0.9);
      border-radius: 9999px;
      border: 1.5px solid transparent;
      pointer-events: none;
      z-index: 99999;
      display: flex;
      align-items: center;
      justify-content: center;
      transform: translate(-50%, -50%);
      transition:
        width 0.35s cubic-bezier(0.23, 1, 0.32, 1),
        height 0.35s cubic-bezier(0.23, 1, 0.32, 1),
        background 0.35s ease,
        border-color 0.35s ease,
        opacity 0.2s ease;
      will-change: transform;
    }

    #cursor-label,
    .cursor-icon {
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
      width: 16px;
      height: 16px;
      position: absolute;
    }

    /* state-view: 80px circle, "View" label */
    #custom-cursor.state-view {
      width: 80px;
      height: 80px;
    }
    #custom-cursor.state-view #cursor-label { opacity: 1; }

    /* state-play: 80px circle, play icon */
    #custom-cursor.state-play {
      width: 80px;
      height: 80px;
    }
    #custom-cursor.state-play #cursor-icon-play { opacity: 1; }

    /* state-explore: 80px circle, "Explore" label */
    #custom-cursor.state-explore {
      width: 80px;
      height: 80px;
    }
    #custom-cursor.state-explore #cursor-label { opacity: 1; }

    /* state-select: 56px circle, "Select" label */
    #custom-cursor.state-select {
      width: 56px;
      height: 56px;
    }
    #custom-cursor.state-select #cursor-label { opacity: 1; }

    /* state-node: 36px outlined ring, no fill, no label */
    #custom-cursor.state-node {
      width: 36px;
      height: 36px;
      background: transparent;
      border-color: rgba(255, 255, 255, 0.6);
    }

    /* state-drag: 64px circle, drag icon */
    #custom-cursor.state-drag {
      width: 64px;
      height: 64px;
    }
    #custom-cursor.state-drag #cursor-icon-drag { opacity: 1; }

    /* state-hidden: invisible (over nav/footer text links) */
    #custom-cursor.state-hidden { opacity: 0; }
  }

  /* Reduced motion: instant size changes, opacity retained */
  @media (prefers-reduced-motion: reduce) {
    #custom-cursor {
      transition: opacity 0.1s ease;
    }
    #cursor-label,
    .cursor-icon {
      transition: none;
    }
  }
  ```

- [ ] **Step 3: Verify in browser**

  Open `index.html`. On a desktop browser with a mouse, the native cursor should be gone and replaced by a small white dot that follows the mouse. The dot should remain 12px everywhere (no states wired yet — that's Task 3).

  Check DevTools Console — no errors.

- [ ] **Step 4: Verify touch / `prefers-reduced-motion` guards**

  In DevTools → Device Toolbar, enable a mobile device (e.g. iPhone 14). The native cursor should reappear (because the `@media (hover: hover)` guard fires). Disable Device Toolbar — small white dot returns.

  In DevTools → Rendering → Emulate CSS media → set `prefers-reduced-motion: reduce`. Move mouse. Dot should still appear and follow; size changes will be instant when states are wired in Task 3.

- [ ] **Step 5: Commit**

  ```bash
  git add index.html
  git commit -m "feat: add custom cursor CSS with all 8 state modifier classes"
  ```

---

### Task 3: Add `initCustomCursor()` JS function

**Files:**
- Modify: `index.html` — inside the `DOMContentLoaded` block, append `initCustomCursor()` call just before the closing `});` of `document.addEventListener('DOMContentLoaded', () => {` (currently at approximately line 3686)

**Interfaces:**
- Consumes: `#custom-cursor` (Task 1), all `.state-*` CSS classes (Task 2)
- Produces: working cursor system with follow loop + all 8 state zones

- [ ] **Step 1: Locate insertion point**

  Find the end of the `DOMContentLoaded` block. It closes at approximately line 3686–3687:
  ```js
        } catch (error) {
          console.error("Magnetic hover setup failed:", error);
        }
      });   // <-- closing of DOMContentLoaded
    </script>
  ```
  Insert the new function definition and its call immediately before that final `});`.

- [ ] **Step 2: Insert `initCustomCursor()`**

  ```js
        // ── Section-aware custom cursor states ───────────────────────────
        function initCustomCursor() {
          const cursor = document.getElementById('custom-cursor');
          const label  = document.getElementById('cursor-label');
          if (!cursor || !label) return;

          // Only run on true pointer devices — mirrors the CSS media query guard
          if (!window.matchMedia('(hover: hover) and (pointer: fine)').matches) return;

          // ── Follow loop: rAF lerp, factor 0.12 ───────────────────────
          let mx = -200, my = -200; // start offscreen to avoid load-flash
          let cx = -200, cy = -200;

          document.addEventListener('mousemove', function(e) {
            mx = e.clientX;
            my = e.clientY;
          });

          (function loop() {
            cx += (mx - cx) * 0.12;
            cy += (my - cy) * 0.12;
            cursor.style.transform = 'translate(' + cx + 'px, ' + cy + 'px) translate(-50%, -50%)';
            requestAnimationFrame(loop);
          })();

          // ── State machine ─────────────────────────────────────────────
          var currentState = '';
          var labelTimeout;

          function setCursorState(state, labelText) {
            labelText = labelText || '';
            if (state === currentState) return;
            currentState = state;

            // Fade out label text, swap content mid-fade, let CSS class re-show it
            label.style.opacity = '0';
            clearTimeout(labelTimeout);
            labelTimeout = setTimeout(function() {
              label.textContent = labelText;
            }, 150);

            cursor.className = state ? ('state-' + state) : '';
          }

          function clearCursorState() {
            setCursorState('');
          }

          // ── Zone wiring ───────────────────────────────────────────────
          // Selectors confirmed against actual DOM in index.html.
          // Pricing: .max-w-6xl.mx-auto > div scopes to pricing grid only
          // (testimonials grid lacks max-w-6xl).
          var zones = [
            { selector: '#horizontal-scroll > div',      state: 'view',    label: 'View'    },
            { selector: '#hero-btn-2',                    state: 'play',    label: ''        },
            { selector: '.dashboard-shell',               state: 'explore', label: 'Explore' },
            { selector: '.max-w-6xl.mx-auto > div',       state: 'select',  label: 'Select'  },
            { selector: '.step-node',                     state: 'node',    label: ''        },
            { selector: '.marquee-container',             state: 'drag',    label: ''        },
            { selector: 'nav a, footer a',                state: 'hidden',  label: ''        },
          ];

          zones.forEach(function(zone) {
            document.querySelectorAll(zone.selector).forEach(function(el) {
              el.addEventListener('mouseenter', function() {
                setCursorState(zone.state, zone.label);
              });
              el.addEventListener('mouseleave', clearCursorState);
            });
          });
        }

        try {
          initCustomCursor();
        } catch (e) {
          console.error('Cursor states failed:', e);
        }
  ```

- [ ] **Step 3: Verify follow behavior in browser**

  Open `index.html`. Move the mouse — the small white dot should smoothly lag behind the pointer with a subtle lerp (~8 frames to reach target). It should not teleport or flash on load.

- [ ] **Step 4: Verify each state zone**

  Hover over each zone and confirm the cursor visually changes:

  | Zone | Expected cursor |
  |---|---|
  | Hover a case study card (`#horizontal-scroll > div`) | Expands to 80px white circle, "View" text appears |
  | Move off card | Shrinks back to 12px dot, text fades |
  | Hover "Watch Demo" button (`#hero-btn-2`) | 80px circle, play triangle icon appears |
  | Hover dashboard shell (`.dashboard-shell`) | 80px circle, "Explore" text |
  | Hover a pricing card (`.max-w-6xl.mx-auto > div`) | 56px circle, "Select" text |
  | Hover a timeline node (`.step-node`) | 36px transparent ring with white border |
  | Hover logo marquee (`.marquee-container`) | 64px circle, drag-arrow icon |
  | Hover a nav link or footer link | Cursor disappears (opacity 0) |
  | Move directly from case study card to pricing card | "View" text fades out, "Select" fades in — no flash |

- [ ] **Step 5: Verify no regressions**

  - Reload the page from scratch. The loader animation must complete normally and the hero entrance must fire.
  - Scroll through the entire page. All GSAP ScrollTrigger animations (services cards, workflow timeline, dashboard pin, bar chart) must animate as before.
  - Open DevTools Console — no new errors. The only cursor-related log that may appear is `'Cursor states failed: ...'` if something breaks, which is the error boundary.

- [ ] **Step 6: Verify `prefers-reduced-motion`**

  In DevTools → Rendering → Emulate CSS media → set `prefers-reduced-motion: reduce`. Hover a case study card. The cursor should jump instantly to 80px (no smooth size tween) but the label should still appear. The lerp follow loop is unaffected (motion preference only controls CSS transitions, not JS animation).

- [ ] **Step 7: Commit**

  ```bash
  git add index.html
  git commit -m "feat: implement section-aware custom cursor with 8 state zones"
  ```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Covered by |
|---|---|
| Default 12px solid dot | Task 2 base `#custom-cursor` styles |
| `state-view` — case study cards, 80px, "View" | Task 2 CSS + Task 3 zone `#horizontal-scroll > div` |
| `state-play` — Watch Demo button, play icon | Task 2 CSS + Task 3 zone `#hero-btn-2` |
| `state-explore` — dashboard shell, "Explore" | Task 2 CSS + Task 3 zone `.dashboard-shell` |
| `state-select` — pricing cards, 56px, "Select" | Task 2 CSS + Task 3 zone `.max-w-6xl.mx-auto > div` |
| `state-node` — step nodes, 36px outlined ring | Task 2 CSS + Task 3 zone `.step-node` |
| `state-drag` — marquee, drag icons | Task 2 CSS + Task 3 zone `.marquee-container` |
| `state-hidden` — nav/footer links, opacity 0 | Task 2 CSS + Task 3 zone `nav a, footer a` |
| rAF lerp follow loop | Task 3 loop |
| CSS transitions for smooth morph | Task 2 `transition` property |
| Label crossfade (150ms fade-out → swap) | Task 3 `setCursorState` timeout |
| `@media (hover: hover)` guard CSS | Task 2 outer media query |
| `@media (hover: hover)` guard JS | Task 3 `matchMedia` check |
| `prefers-reduced-motion` | Task 2 override block |
| `try/catch` error boundary | Task 3 wrapping `try { initCustomCursor() }` |
| `aria-hidden="true"` | Task 1 HTML attribute |
| Centralized `setCursorState()` + zones array | Task 3 — single function, no one-off blocks |
| No new colors | All states use white/black/gray only |

**Placeholder scan:** None found — all steps have exact code.

**Type/name consistency:** `setCursorState`, `clearCursorState`, `currentState`, `labelTimeout`, `zones` — all names consistent across Task 3. CSS class names (`state-view`, `state-play`, `state-explore`, `state-select`, `state-node`, `state-drag`, `state-hidden`) match exactly between Task 2 CSS and Task 3 JS string concatenation (`'state-' + state`).
