# Scroll-Sequence Recipe

**Trigger:** user points at a folder of image frames and says something like "make this folder into scroll."

**Deliverable:** one self-contained HTML file (GSAP + ScrollTrigger, canvas image sequence scrubbed by scroll — the Apple AirPods technique) plus a `frames/` folder next to it.

A known-good reference implementation lives at `scroll-sequence.html` in this repo. For chaining MULTIPLE clips seamlessly on one pinned canvas with per-chapter caption overlays (fade in/out windows tied to scroll progress), use `scroll-combined.html` as the reference instead — its `SEGMENTS` and `CAPTIONS` arrays are the only parts to edit.

---

## Step 1 — Inspect the frame folder

Detect, don't assume:

1. **Count** the image files (jpg/png/webp).
2. **Filename pattern** — find the common prefix, the zero-padding width, the start index (0 or 1), and the extension. Example: `ezgif-frame-001.jpg` → prefix `ezgif-frame-`, pad 3, starts at 1, `.jpg`.
3. **Dimensions** of one frame (PowerShell: `System.Drawing.Image::FromFile`). Only used for sanity — the renderer cover-fits any aspect ratio.
4. Sanity-check: file count matches the numeric range with no gaps. If gaps exist, tell the user instead of guessing.

## Step 2 — Copy frames into the project

Copy all frames into a `frames/` folder next to where the HTML will live. The HTML references them relatively (`frames/<name>`), so the pair ships together.

## Step 3 — Generate the HTML

Copy the full template at the bottom of this file into `<name>.html` and change **only** the `CONFIG` block:

```js
var CONFIG = {
  frameCount: <detected count>,
  framePath: function (i) {
    return "frames/<prefix>" + String(i + <start index>).padStart(<pad width>, "0") + ".<ext>";
  },
  scrollLength: 3000,   // px of pinned scroll; bigger = slower playback. ~12px per frame is a good floor.
  scrub: 0.7,           // seconds of catch-up smoothing
  maxDPR: 2             // devicePixelRatio cap for mobile sharpness vs fill-rate
};
```

Also update the `<title>`, the two copy sections (headline text), and the `<noscript>` img src to the first frame's real filename.

**Do not touch anything else.** The template already handles: preload with progress overlay, nearest-loaded-frame fallback while loading, rAF-deduped rendering, DPR-aware cover-fit centered drawing, pin + scrub via ScrollTrigger, resize/orientation re-render, `<noscript>`/CDN-failure static fallback.

### GSAP scripts + SRI

Keep these tags verbatim (hashes verified for gsap@3.12.5 from raw bytes):

```html
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/gsap.min.js"
        integrity="sha384-g4NTh/Iv5PPU4xPyhEWqPcwtNXOvdaDI8LLnyYfyNZOjKJeYQyjzQ9X5275eBjpt"
        crossorigin="anonymous"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/ScrollTrigger.min.js"
        integrity="sha384-Z3REaz79l2IaAZqJsSABtTbhjgOUYyV3p90XNnAPCSHg3EMTz1fouunq9WZRtj3d"
        crossorigin="anonymous"></script>
```

If you bump the GSAP version, recompute both sha384 hashes from the downloaded raw bytes (`Get-FileHash -Algorithm SHA384`, hex → base64). Never guess an SRI hash — a wrong hash blocks the script entirely.

## Step 4 — Verify in browser (required before claiming done)

Serve over HTTP — the browser pane blocks `file://` navigation from tools. Use `preview_start` with the `static` config in `.claude/launch.json` (python http.server on port 8123), then navigate the **main** tab to `http://localhost:8123/<name>.html`. Secondary preview tabs can sit backgrounded (`document.hidden: true`), which makes screenshots return black even though the canvas painted — if a screenshot looks black, check `document.hidden` and verify paint with `getImageData` before suspecting the page.

Confirm:

- [ ] First frame paints; loader bar fills and fades out
- [ ] Scrolling down pins the canvas and advances frames (screenshot before/after scroll — image must change)
- [ ] `ScrollTrigger.getAll()[0]` shows `isActive: true` mid-sequence and `progress` moving
- [ ] Mobile preset (375×812): image cover-fits, centered, sharp
- [ ] Scroll to bottom: section unpins into the outro copy
- [ ] Zero console errors

## Gotchas learned building the reference

- **Progress bar must animate `transform: scaleX()`, not `width`** — layout-property animation fails the design hook and thrashes layout.
- **Renderer never assumes a frame is loaded.** `nearestLoadedFrame()` scans outward from the requested index; `onerror` still increments the loaded counter so the loader can't hang on a 404.
- **Only repaint when the integer frame changes**, and route through `requestAnimationFrame` — scrub emits many updates per painted frame.
- **`snap: "frame"` + `ease: "none"`** on the tween keeps frame mapping linear and integer-aligned.
- **`100svh` with `100vh` fallback** on the pinned section avoids mobile address-bar jump.
- Frames stay relative (`frames/...`) so the HTML + folder can be dropped into any site together.

---

## Template

Everything below is the full reference file. Copy verbatim, then apply only the Step-3 edits.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
<title>Scroll Sequence</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }

  html { scroll-behavior: auto; }

  body {
    background: #000;
    color: #f5f5f7;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    overflow-x: hidden;
  }

  section.copy {
    min-height: 100vh;
    min-height: 100svh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    padding: 0 24px;
    gap: 16px;
  }

  section.copy h1 {
    font-size: clamp(2.2rem, 6vw, 4.5rem);
    font-weight: 700;
    letter-spacing: -0.02em;
  }

  section.copy p {
    font-size: clamp(1rem, 2.2vw, 1.35rem);
    color: #a1a1a6;
    max-width: 40ch;
  }

  .scroll-hint {
    margin-top: 24px;
    font-size: 0.85rem;
    color: #6e6e73;
    animation: hint-bob 2s ease-in-out infinite;
  }

  @keyframes hint-bob {
    0%, 100% { transform: translateY(0); opacity: 0.6; }
    50%      { transform: translateY(6px); opacity: 1; }
  }

  /* The pinned stage. ScrollTrigger pins this element while the sequence plays. */
  #sequence {
    position: relative;
    height: 100vh;
    height: 100svh;
    overflow: hidden;
  }

  #sequence canvas {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    display: block;
  }

  /* Loading fallback overlay */
  #loader {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 14px;
    background: rgba(0, 0, 0, 0.55);
    backdrop-filter: blur(6px);
    -webkit-backdrop-filter: blur(6px);
    transition: opacity 0.5s ease;
    z-index: 2;
  }

  #loader.done { opacity: 0; pointer-events: none; }

  #loader .label {
    font-size: 0.8rem;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: #a1a1a6;
    font-variant-numeric: tabular-nums;
  }

  #loader .bar {
    width: min(220px, 50vw);
    height: 2px;
    background: rgba(255, 255, 255, 0.15);
    border-radius: 2px;
    overflow: hidden;
  }

  #loader .bar i {
    display: block;
    height: 100%;
    width: 100%;
    background: #f5f5f7;
    border-radius: 2px;
    transform: scaleX(0);
    transform-origin: left center;
    transition: transform 0.2s ease;
  }

  /* Static fallback if JS or GSAP never arrives */
  #sequence noscript img,
  #sequence .static-fallback {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
</style>
</head>
<body>

  <section class="copy">
    <h1>Scroll to play.</h1>
    <p>An image sequence rendered to canvas, driven frame by frame by your scroll position.</p>
    <div class="scroll-hint">&darr; scroll</div>
  </section>

  <section id="sequence" aria-label="Scroll-driven animation">
    <canvas id="sequence-canvas"></canvas>
    <div id="loader" aria-hidden="true">
      <div class="bar"><i id="loader-fill"></i></div>
      <div class="label" id="loader-label">Loading 0%</div>
    </div>
    <noscript><img src="frames/ezgif-frame-001.jpg" alt="Animation still"></noscript>
  </section>

  <section class="copy">
    <h1>That&rsquo;s the technique.</h1>
    <p>Pinned canvas, preloaded frames, scrubbed timeline &mdash; the same approach Apple uses on its product pages.</p>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/gsap.min.js"
          integrity="sha384-g4NTh/Iv5PPU4xPyhEWqPcwtNXOvdaDI8LLnyYfyNZOjKJeYQyjzQ9X5275eBjpt"
          crossorigin="anonymous"></script>
  <script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/ScrollTrigger.min.js"
          integrity="sha384-Z3REaz79l2IaAZqJsSABtTbhjgOUYyV3p90XNnAPCSHg3EMTz1fouunq9WZRtj3d"
          crossorigin="anonymous"></script>

  <script>
  (function () {
    "use strict";

    /* ============================================================
       CONFIG — the only part you need to touch to reuse this.
       ============================================================ */
    var CONFIG = {
      frameCount: 240,
      framePath: function (i) {
        // frames/ezgif-frame-001.jpg ... frames/ezgif-frame-240.jpg
        return "frames/ezgif-frame-" + String(i + 1).padStart(3, "0") + ".jpg";
      },
      // How much scroll distance (px) the sequence occupies while pinned.
      // Bigger = slower, more cinematic playback.
      scrollLength: 3000,
      // Scrub smoothing in seconds. Canvas catches up to the scrollbar over
      // this duration, which irons out choppy wheel/trackpad deltas.
      scrub: 0.7,
      // Cap devicePixelRatio so 3x/4x phones don't burn fill-rate for
      // sharpness nobody can see.
      maxDPR: 2
    };

    var canvas = document.getElementById("sequence-canvas");
    var ctx = canvas.getContext("2d");
    var loader = document.getElementById("loader");
    var loaderFill = document.getElementById("loader-fill");
    var loaderLabel = document.getElementById("loader-label");

    var frames = new Array(CONFIG.frameCount);
    var frameLoaded = new Array(CONFIG.frameCount).fill(false);
    var loadedCount = 0;
    var currentFrame = 0;
    var renderQueued = false;

    /* ============================================================
       PRELOAD
       Frame 0 is requested first so the stage never sits black;
       the rest stream in behind it. Every arrival updates the
       progress bar, and if the user is already parked on a frame
       that just landed, we repaint immediately.
       ============================================================ */
    function preload() {
      for (var i = 0; i < CONFIG.frameCount; i++) {
        (function (index) {
          var img = new Image();
          img.decoding = "async";
          img.onload = function () {
            frameLoaded[index] = true;
            loadedCount++;
            updateLoader();
            if (index === currentFrame || index === 0) requestRender();
          };
          img.onerror = function () {
            // Count errors as "done" so the loader can't hang forever;
            // nearestLoadedFrame() papers over the hole at render time.
            loadedCount++;
            updateLoader();
          };
          img.src = CONFIG.framePath(index);
          frames[index] = img;
        })(i);
      }
    }

    function updateLoader() {
      var pct = Math.round((loadedCount / CONFIG.frameCount) * 100);
      loaderFill.style.transform = "scaleX(" + loadedCount / CONFIG.frameCount + ")";
      loaderLabel.textContent = "Loading " + pct + "%";
      if (loadedCount >= CONFIG.frameCount) {
        loader.classList.add("done");
        requestRender();
      }
    }

    /* Graceful fallback while frames are still arriving: if the frame
       the scrollbar asks for isn't decoded yet, draw the closest one
       that is. The sequence stays visibly interactive during load and
       self-heals as gaps fill in. */
    function nearestLoadedFrame(target) {
      if (frameLoaded[target]) return target;
      for (var d = 1; d < CONFIG.frameCount; d++) {
        if (target - d >= 0 && frameLoaded[target - d]) return target - d;
        if (target + d < CONFIG.frameCount && frameLoaded[target + d]) return target + d;
      }
      return -1; // nothing loaded yet
    }

    /* ============================================================
       CANVAS SIZING + DRAW
       Backing store is sized in device pixels (capped at maxDPR)
       so the image stays sharp on retina/mobile screens, and each
       frame is drawn "cover"-style: scaled to fill and centered,
       like CSS object-fit: cover.
       ============================================================ */
    function resizeCanvas() {
      var rect = canvas.parentElement.getBoundingClientRect();
      var dpr = Math.min(window.devicePixelRatio || 1, CONFIG.maxDPR);
      canvas.width = Math.round(rect.width * dpr);
      canvas.height = Math.round(rect.height * dpr);
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
      ctx.imageSmoothingEnabled = true;
      ctx.imageSmoothingQuality = "high";
      drawFrame(currentFrame);
    }

    function drawFrame(index) {
      var usable = nearestLoadedFrame(index);
      if (usable === -1) return;
      var img = frames[usable];
      var cw = canvas.width / Math.min(window.devicePixelRatio || 1, CONFIG.maxDPR);
      var ch = canvas.height / Math.min(window.devicePixelRatio || 1, CONFIG.maxDPR);
      var scale = Math.max(cw / img.naturalWidth, ch / img.naturalHeight);
      var dw = img.naturalWidth * scale;
      var dh = img.naturalHeight * scale;
      ctx.clearRect(0, 0, cw, ch);
      ctx.drawImage(img, (cw - dw) / 2, (ch - dh) / 2, dw, dh);
    }

    /* Renders are funneled through requestAnimationFrame so rapid
       scrub updates collapse into one paint per display refresh. */
    function requestRender() {
      if (renderQueued) return;
      renderQueued = true;
      requestAnimationFrame(function () {
        renderQueued = false;
        drawFrame(currentFrame);
      });
    }

    /* ============================================================
       SCROLL → FRAME MAPPING (GSAP + ScrollTrigger)
       A proxy object's `frame` value is tweened 0 → N-1 across the
       pinned scroll distance. `scrub` smooths it, `snap: "frame"`
       keeps it on whole frames, and onUpdate repaints only when the
       integer frame actually changes.
       ============================================================ */
    function initScroll() {
      gsap.registerPlugin(ScrollTrigger);

      var seq = { frame: 0 };

      gsap.to(seq, {
        frame: CONFIG.frameCount - 1,
        ease: "none",
        snap: "frame",
        scrollTrigger: {
          trigger: "#sequence",
          start: "top top",
          end: "+=" + CONFIG.scrollLength,
          pin: true,
          scrub: CONFIG.scrub,
          anticipatePin: 1,
          invalidateOnRefresh: true
        },
        onUpdate: function () {
          var f = Math.round(seq.frame);
          if (f !== currentFrame) {
            currentFrame = f;
            requestRender();
          }
        }
      });
    }

    /* ============================================================
       BOOT
       ============================================================ */
    resizeCanvas();
    preload();

    if (window.gsap && window.ScrollTrigger) {
      initScroll();
    } else {
      // CDN blocked/offline: no pin, no scrub — just show the first
      // frame as a static image so the page still reads correctly.
      loader.classList.add("done");
    }

    var resizeTimer;
    window.addEventListener("resize", function () {
      clearTimeout(resizeTimer);
      resizeTimer = setTimeout(function () {
        resizeCanvas();
        if (window.ScrollTrigger) ScrollTrigger.refresh();
      }, 150);
    });
    window.addEventListener("orientationchange", function () {
      setTimeout(resizeCanvas, 300);
    });
  })();
  </script>
</body>
</html>
```
