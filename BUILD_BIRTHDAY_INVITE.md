# BUILD_BIRTHDAY_INVITE.md

## Master Reference — Immersive Birthday Invitation (Kid or Adult Themes)

> A comprehensive, production-grade guide for building fully responsive, multi-section birthday invitation pages with video intros, animated canvas layers, RSVP-to-WhatsApp integration, and premium scroll-reveal effects.

---

## Table of Contents

1. [Technical Architecture & Setup](#1-technical-architecture--setup)
2. [Video Intro Logic (`#intro-layer`)](#2-video-intro-logic-intro-layer)
3. [Core UI Elements & Animations](#3-core-ui-elements--animations)
4. [Content Sections (Adaptable for Kid/Adult)](#4-content-sections-adaptable-for-kidadult)
5. [RSVP Form & WhatsApp Integration](#5-rsvp-form--whatsapp-integration)
6. [Strict Footer Implementation](#6-strict-footer-implementation)
7. [Build Checklist](#7-build-checklist)

---

## 1. Technical Architecture & Setup

### 1.1 — Separation of Concerns

> **CRITICAL RULE:** Every project MUST split code into three separate files. A single monolithic HTML file is NOT acceptable.

```
project-root/
├── index.html          ← Markup & structure only
├── style.css           ← All styles, variables, keyframes, media queries
├── script.js           ← All interactivity, observers, form logic
├── images/             ← Local assets (banners, thumbnails, decorations)
└── audio/              ← Background music files
```

**`index.html`** links them in the `<head>` and before `</body>`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport"
        content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
  <title>Birthday Invitation</title>

  <!-- Fonts -->
  <link href="https://fonts.googleapis.com/css2?family=Fredoka+One&family=Quicksand:wght@400;500;600;700&family=Pacifico&display=swap"
        rel="stylesheet">

  <!-- Preload critical first-paint assets -->
  <link rel="preload" href="path/to/intro-mobile.mp4"  as="video" type="video/mp4"
        media="(max-width: 767px)" crossorigin="anonymous">
  <link rel="preload" href="path/to/intro-desktop.mp4" as="video" type="video/mp4"
        media="(min-width: 768px)" crossorigin="anonymous">
  <link rel="preload" href="images/hero-banner-mobile.avif"  as="image"
        media="(max-width: 767px)" fetchpriority="high">
  <link rel="preload" href="images/hero-banner-desktop.jpg"  as="image"
        media="(min-width: 768px)" fetchpriority="high">

  <link rel="icon" href="data:,">

  <!-- Stylesheet -->
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <!-- ... markup ... -->

  <!-- Vendor scripts (Embla Carousel) -->
  <script src="https://unpkg.com/embla-carousel/embla-carousel.umd.js"></script>
  <script src="https://unpkg.com/embla-carousel-autoplay/embla-carousel-autoplay.umd.js"></script>

  <!-- Application script -->
  <script src="script.js"></script>
</body>
</html>
```

### 1.2 — Viewport Handling

The `<meta name="viewport">` tag is **non-negotiable**. It MUST be:

```html
<meta name="viewport"
      content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
```

| Attribute            | Purpose                                                     |
|----------------------|-------------------------------------------------------------|
| `width=device-width` | Matches CSS pixel width to device screen                    |
| `initial-scale=1.0`  | Prevents zoom on page load                                  |
| `maximum-scale=1.0`  | Disables pinch-zoom (preserves immersive feel)              |
| `user-scalable=no`   | Redundant safeguard for older WebViews                      |
| `viewport-fit=cover` | Extends rendering into notch / dynamic island safe areas    |

### 1.3 — Window Height Calculation (`setActualVh`)

Mobile browser address bars resize `100vh` dynamically. This causes jumpy layouts. The **very first** executable line in `script.js` MUST be the `setActualVh()` function:

```js
// ──────────────────────────────────────────────────────────────
// MUST BE THE FIRST EXECUTABLE CODE IN script.js
// ──────────────────────────────────────────────────────────────
(function setActualVh() {
  const vh = (window.visualViewport?.height ?? window.innerHeight) * 0.01;
  document.documentElement.style.setProperty('--actual-vh', `${vh}px`);
})();

// Re-calculate on resize / orientation change
['resize', 'orientationchange'].forEach(evt =>
  window.addEventListener(evt, () => {
    const vh = (window.visualViewport?.height ?? window.innerHeight) * 0.01;
    document.documentElement.style.setProperty('--actual-vh', `${vh}px`);
  })
);

// Also listen on visualViewport resize (handles mobile keyboard / toolbar changes)
if (window.visualViewport) {
  window.visualViewport.addEventListener('resize', () => {
    const vh = window.visualViewport.height * 0.01;
    document.documentElement.style.setProperty('--actual-vh', `${vh}px`);
  });
}
```

Use it in CSS wherever you would reach for `100vh`:

```css
/* Instead of: min-height: 100vh; */
.hero-sec {
  min-height: calc(var(--actual-vh, 1vh) * 100);
}
```

### 1.4 — Scroll Restoration & Locking

Immediately after `setActualVh()`, add:

```js
// Prevent browser scroll restoration (back-button, refresh)
if (history.scrollRestoration) {
  history.scrollRestoration = 'manual';
}
window.scrollTo(0, 0);
```

### 1.5 — Body State (Overflow Locking)

**CSS** — the body starts locked:

```css
body {
  overflow: hidden;              /* Scroll is LOCKED until intro finishes */
  font-family: 'Quicksand', sans-serif;
  -webkit-font-smoothing: antialiased;
  background: #111;
}

body.unlocked {
  overflow: auto;                /* Scroll is UNLOCKED */
  background: #e8ded3;          /* Reveal warm bg behind the app container */
}
```

**JS** — the `.unlocked` class is added ONLY after the intro sequence finishes:

```js
function finishIntro() {
  // ... (fade logic, covered in §2) ...
  document.body.classList.add('unlocked');
}
```

### 1.6 — Icons

**All icons MUST be inline SVGs.** Do NOT use icon fonts (FontAwesome, Material Icons) or external image files for UI icons.

Rationale:
- Zero additional HTTP requests
- CSS-controllable (`stroke`, `fill`, `width`, `height`)
- No FOUT (flash of unstyled text) from icon font loading
- Accessible via `aria-label` on the parent element

```html
<!-- ✅ Correct: Inline SVG -->
<svg viewBox="0 0 24 24" width="28" height="28" fill="none"
     stroke="currentColor" stroke-width="2"
     stroke-linecap="round" stroke-linejoin="round">
  <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"></polygon>
  <path d="M19.07 4.93a10 10 0 0 1 0 14.14M15.54 8.46a5 5 0 0 1 0 7.07"></path>
</svg>

<!-- ❌ Wrong: External icon font -->
<i class="fa fa-volume-up"></i>
```

---

## 2. Video Intro Logic (`#intro-layer`)

### 2.1 — HTML Structure

The intro layer sits as a **fixed, full-screen overlay** at the highest z-index. It contains the video and a tap prompt:

```html
<!-- Inside #app or directly in <body> -->
<div id="intro-layer">
  <video id="intro-video"
         playsinline webkit-playsinline x5-playsinline
         disablePictureInPicture
         fetchpriority="high"
         preload="auto"
         poster="images/intro-poster.webp">
    <!-- Sources set dynamically via JS for mobile/desktop -->
  </video>

  <div class="tap-overlay">
    <span class="tap-text" id="tap-text">Tap to Open</span>
  </div>

  <!-- Skip Intro Button (top-right corner) -->
  <button id="skip-intro-btn" class="skip-intro-btn" aria-label="Skip intro video">
    Skip
    <svg viewBox="0 0 24 24" width="14" height="14" fill="none"
         stroke="currentColor" stroke-width="2.5"
         stroke-linecap="round" stroke-linejoin="round">
      <polyline points="13 17 18 12 13 7"></polyline>
      <polyline points="6 17 11 12 6 7"></polyline>
    </svg>
  </button>
</div>
```

### 2.2 — CSS

```css
#intro-layer {
  position: fixed;
  top: 0; left: 0;
  width: 100vw;
  height: calc(var(--actual-vh, 1vh) * 100);
  z-index: 1000;
  background: #000;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  transition: opacity 1.2s ease-in-out;
  overflow: hidden;
  cursor: pointer;
}

#intro-video {
  position: absolute;
  top: 0; left: 50%;
  transform: translateX(-50%);
  width: 100%; height: 100%;
  object-fit: cover;
  pointer-events: none;           /* Clicks pass through to #intro-layer */
}

.tap-overlay {
  position: relative;
  z-index: 1010;
  text-align: center;
  animation: tapPulse 2s infinite;
  padding: 40px;
  pointer-events: none;
  margin-top: 40px;
}

.tap-text {
  font-family: 'Quicksand', sans-serif;
  font-weight: 700;
  color: #fff;
  font-size: 1.2rem;
  letter-spacing: 2px;
  text-transform: uppercase;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.8);
  background: rgba(0, 0, 0, 0.4);
  padding: 10px 20px;
  border-radius: 30px;
}

@keyframes tapPulse {
  0%, 100% { transform: scale(1); }
  50%      { transform: scale(1.05); }
}

/* ── Skip Intro Button ── */
.skip-intro-btn {
  position: absolute;
  top: max(16px, env(safe-area-inset-top, 16px));
  right: 16px;
  z-index: 1020;
  display: none;                          /* Hidden until video starts playing */
  align-items: center;
  gap: 4px;
  padding: 8px 16px;
  background: rgba(0, 0, 0, 0.45);
  backdrop-filter: blur(6px);
  -webkit-backdrop-filter: blur(6px);
  color: #fff;
  font-family: 'Quicksand', sans-serif;
  font-size: 0.8rem;
  font-weight: 700;
  letter-spacing: 1px;
  text-transform: uppercase;
  border: 1.5px solid rgba(255, 255, 255, 0.35);
  border-radius: 20px;
  cursor: pointer;
  transition: background 0.2s ease, transform 0.15s ease;
}

.skip-intro-btn:hover {
  background: rgba(0, 0, 0, 0.65);
}

.skip-intro-btn:active {
  transform: scale(0.95);
}

.skip-intro-btn.visible {
  display: flex;
}
```

### 2.3 — JavaScript Logic

**Key Rules:**
- Exactly **one tap** to play. After the first tap, all subsequent taps are ignored.
- The video **cannot be paused** by the user (but can be **skipped** via a dedicated button).
- A **Skip button** appears in the top-right corner once the video begins playing. Tapping it calls `finishIntro()` immediately.
- Scrolling **remains locked** until the video ends or is skipped.
- The fade-out begins **before the video ends** (e.g., 1.2s before) so the Hero section is visible behind the dissolve — never fade to a black screen.
- After the fade completes: add `.unlocked`, show Music FAB, show Scroll Indicator.

```js
const introLayer = document.getElementById('intro-layer');
const introVideo = document.getElementById('intro-video');
const tapText    = document.getElementById('tap-text');
const bgAudio    = document.getElementById('bg-music');
const skipBtn    = document.getElementById('skip-intro-btn');
let isOpened     = false;

// ── Responsive video source selection ────────────────────────
const isMobileView = () => {
  if (window.matchMedia('(orientation: landscape)').matches && window.innerWidth >= 480) return false;
  return window.innerWidth < 768;
};

const VIDEO_MOBILE  = 'path/to/intro-mobile.mp4';
const VIDEO_DESKTOP = 'path/to/intro-desktop.mp4';
let currentVideoUrl = isMobileView() ? VIDEO_MOBILE : VIDEO_DESKTOP;

introVideo.poster = isMobileView() ? 'images/poster-mobile.webp' : 'images/poster-desktop.webp';
introVideo.src    = currentVideoUrl;
introVideo.load();

// ── Video readiness detection ────────────────────────────────
// Page loader stays visible until the intro video is buffered.
const pageLoader = document.getElementById('page-loader');
let loaderHidden = false;

function hidePageLoader() {
  if (loaderHidden) return;
  loaderHidden = true;
  if (pageLoader) {
    pageLoader.classList.add('fade-out');
    setTimeout(() => { pageLoader.style.display = 'none'; }, 520);
  }
}

if (introVideo) {
  let rafPollId = null;

  function onVideoReady() {
    if (rafPollId) { cancelAnimationFrame(rafPollId); rafPollId = null; }
    introVideo.removeEventListener('canplaythrough', onVideoReady);
    introVideo.removeEventListener('loadeddata', onVideoReady);
    try { introVideo.pause(); introVideo.currentTime = 0; } catch (e) {}
    hidePageLoader();
  }

  introVideo.addEventListener('canplaythrough', onVideoReady, { once: true });
  introVideo.addEventListener('loadeddata', onVideoReady, { once: true });
  introVideo.addEventListener('error', () => hidePageLoader(), { once: true });

  // rAF polling — critical fallback for iOS Safari
  function pollReadyState() {
    if (introVideo.readyState >= 3) {
      onVideoReady();
    } else {
      rafPollId = requestAnimationFrame(pollReadyState);
    }
  }
  rafPollId = requestAnimationFrame(pollReadyState);
}

// Safety timeouts
setTimeout(() => {
  if (!loaderHidden && introVideo && introVideo.readyState >= 2) hidePageLoader();
}, 6000);
setTimeout(hidePageLoader, 12000);

// ── Safe play helper (must be inside a click handler for Safari) ──
function safePlayVideo() {
  if (!introVideo) return;
  introVideo.muted  = false;
  introVideo.volume = 1.0;
  const p = introVideo.play();
  if (p !== undefined) {
    p.catch(() => {
      introVideo.play().catch(err => console.log('Final video play failure:', err));
    });
  }
}

// ── One-tap handler ──────────────────────────────────────────
introLayer.addEventListener('click', () => {
  if (isOpened) return;            // Ignore all taps after the first
  isOpened = true;
  tapText.style.display = 'none';

  // Unlock audio context on first interaction (required by mobile browsers)
  bgAudio.volume = 0;
  bgAudio.play().then(() => {
    bgAudio.pause();
    bgAudio.currentTime = 0;
  }).catch(() => {});

  // Begin fade 1.2s before the video ends for a seamless transition
  introVideo.ontimeupdate = () => {
    if (introVideo.duration && introVideo.currentTime >= introVideo.duration - 1.2) {
      finishIntro();
    }
  };
  introVideo.onended = finishIntro;

  // Play video WITH audio — synchronous call inside click = Safari allows it
  safePlayVideo();

  // Show the Skip button now that the video is playing
  if (skipBtn) skipBtn.classList.add('visible');

  // Hard fallback if video stalls or is ultra-short
  setTimeout(() => {
    if (introLayer.style.display !== 'none') finishIntro();
  }, 15000);
});

// ── Skip Intro button handler ────────────────────────────────
if (skipBtn) {
  skipBtn.addEventListener('click', (e) => {
    e.stopPropagation();             // Don't re-trigger the intro-layer click
    finishIntro();
  });
}

// ── Finish Intro (fade + unlock) ─────────────────────────────
let introFinished = false;

function finishIntro() {
  if (introFinished) return;
  introFinished = true;

  // Hide the Skip button immediately
  if (skipBtn) skipBtn.classList.remove('visible');

  // ── 1. Fade in background music ──
  bgAudio.volume = 0;
  bgAudio.play().catch(() => {});
  let vol = 0;
  const fadeAudio = setInterval(() => {
    if (vol < 0.4) {
      vol += 0.05;
      bgAudio.volume = Math.min(vol, 0.4);
    } else {
      clearInterval(fadeAudio);
    }
  }, 200);

  // ── 2. Reveal main content BEHIND the intro layer ──
  const mainContent = document.getElementById('main-content');
  if (mainContent) {
    mainContent.style.display = 'block';
    setTimeout(() => { mainContent.style.opacity = '1'; }, 50);
  }

  // ── 3. Fade out intro layer (hero visible behind it — NOT black) ──
  if (introLayer) {
    introLayer.style.pointerEvents = 'none';
    introLayer.style.opacity = '0';                  // CSS transition: 1.2s

    setTimeout(() => {
      introLayer.style.display = 'none';
      try { introVideo.pause(); } catch (e) {}

      // ── 4. Unlock body scroll ──
      document.body.classList.add('unlocked');

      // ── 5. Show Music FAB & Scroll Indicator ──
      const audioCtrl = document.getElementById('audio-ctrl');
      if (audioCtrl) audioCtrl.style.opacity = '1';

      const scrollInd = document.getElementById('scroll-indicator');
      if (scrollInd) scrollInd.classList.add('visible');

      // ── 6. NOW init scroll reveal (NEVER before .unlocked) ──
      initScrollReveal();
    }, 1200);
  }
}
```

### 2.4 — Responsive Video Swapping

When the viewport size changes before the user taps (e.g., device rotation), swap the video source:

```js
let rsz;
window.addEventListener('resize', () => {
  clearTimeout(rsz);
  rsz = setTimeout(() => {
    if (isOpened) return;
    const target = isMobileView() ? VIDEO_MOBILE : VIDEO_DESKTOP;
    if (currentVideoUrl !== target) {
      currentVideoUrl  = target;
      introVideo.poster = isMobileView() ? 'images/poster-mobile.webp' : 'images/poster-desktop.webp';
      introVideo.src    = target;
      introVideo.load();
    }
  }, 120);
});

window.addEventListener('orientationchange', () => {
  const sync = () => {
    if (isOpened) return;
    const target = isMobileView() ? VIDEO_MOBILE : VIDEO_DESKTOP;
    if (currentVideoUrl !== target) {
      currentVideoUrl  = target;
      introVideo.poster = isMobileView() ? 'images/poster-mobile.webp' : 'images/poster-desktop.webp';
      introVideo.src    = target;
      introVideo.load();
    }
  };
  sync();
  setTimeout(sync, 150);
  setTimeout(sync, 350);
});
```

---

## 3. Core UI Elements & Animations

### 3.1 — Music FAB (Floating Action Button)

The FAB uses `env(safe-area-inset-bottom)` for iOS home indicator clearance and a `max()` fallback for Android gesture navigation:

**CSS:**

```css
.audio-ctrl {
  position: fixed;
  bottom: max(72px, calc(env(safe-area-inset-bottom, 0px) + 38px));
  right: 25px;
  width: 56px;
  height: 56px;
  background: var(--accent);
  color: #fff;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  z-index: 100;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
  border: 3px solid #fff;
  opacity: 0;                     /* Hidden until intro finishes */
  transition: opacity 0.6s ease;
}

/* Desktop override */
@media (min-width: 768px) {
  .audio-ctrl {
    right: 40px;
    width: 72px;
    height: 72px;
  }
  .audio-ctrl svg { width: 36px; height: 36px; }
}
```

**HTML — inline SVG icons (muted & unmuted):**

```html
<div id="audio-ctrl" class="audio-ctrl" aria-label="Toggle music">
  <!-- Muted Icon -->
  <svg id="icon-muted" viewBox="0 0 24 24" width="28" height="28"
       fill="none" stroke="currentColor" stroke-width="2"
       stroke-linecap="round" stroke-linejoin="round"
       style="display:none;">
    <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"></polygon>
    <line x1="23" y1="9" x2="17" y2="15"></line>
    <line x1="17" y1="9" x2="23" y2="15"></line>
  </svg>

  <!-- Unmuted Icon -->
  <svg id="icon-unmuted" viewBox="0 0 24 24" width="28" height="28"
       fill="none" stroke="currentColor" stroke-width="2"
       stroke-linecap="round" stroke-linejoin="round">
    <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"></polygon>
    <path d="M19.07 4.93a10 10 0 0 1 0 14.14M15.54 8.46a5 5 0 0 1 0 7.07"></path>
  </svg>
</div>

<audio id="bg-music" loop>
  <source src="audio/birthday-music.mp3" type="audio/mpeg">
</audio>
```

**JS — Toggle logic:**

```js
const audioCtrl  = document.getElementById('audio-ctrl');
const iconMuted  = document.getElementById('icon-muted');
const iconUnmuted = document.getElementById('icon-unmuted');
const bgAudio    = document.getElementById('bg-music');

audioCtrl.addEventListener('click', () => {
  if (bgAudio.muted) {
    bgAudio.muted = false;
    bgAudio.play().catch(() => {});
    iconMuted.style.display   = 'none';
    iconUnmuted.style.display = 'block';
  } else {
    bgAudio.muted = true;
    iconMuted.style.display   = 'block';
    iconUnmuted.style.display = 'none';
  }
});
```

### 3.2 — Canvas Petal / Confetti Animation

A lightweight canvas layer for ambient petal-fall (kids) or confetti-shimmer (adults).

**HTML:**

```html
<canvas id="petal-canvas"></canvas>
```

**CSS:**

```css
#petal-canvas {
  position: fixed;
  top: 0; left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 50;
}
```

**JS — Performance-safe canvas loop:**

```js
function initPetalCanvas() {
  // ── Respect accessibility: skip animation for reduced-motion users ──
  const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
  if (prefersReducedMotion) return;

  const canvas = document.getElementById('petal-canvas');
  if (!canvas) return;
  const ctx = canvas.getContext('2d');

  function resizeCanvas() {
    canvas.width  = window.innerWidth;
    canvas.height = window.visualViewport?.height ?? window.innerHeight;
  }
  resizeCanvas();
  window.addEventListener('resize', resizeCanvas);

  // ── Particle count: capped between 10 and 30 for performance ──
  const area = canvas.width * canvas.height;
  const PARTICLE_COUNT = Math.max(10, Math.min(30, Math.floor(area / 25000)));

  const particles = [];

  // ── Particle factory ──
  // Customize colors for kid (pastels) vs adult (gold/champagne) themes
  const COLORS_KID   = ['#FFB6C1', '#FFD700', '#87CEEB', '#98FB98', '#DDA0DD'];
  const COLORS_ADULT = ['#D4AF37', '#C9B037', '#E8D5B7', '#FFF8E7', '#BFA76A'];

  // Switch palette based on a data attribute or JS config:
  const isKidTheme = document.documentElement.dataset.theme !== 'adult';
  const COLORS = isKidTheme ? COLORS_KID : COLORS_ADULT;

  for (let i = 0; i < PARTICLE_COUNT; i++) {
    particles.push({
      x:        Math.random() * canvas.width,
      y:        Math.random() * canvas.height,
      size:     Math.random() * 6 + 3,
      speedY:   Math.random() * 0.8 + 0.3,
      speedX:   Math.random() * 0.6 - 0.3,
      opacity:  Math.random() * 0.6 + 0.3,
      rotation: Math.random() * 360,
      rotSpeed: Math.random() * 2 - 1,
      color:    COLORS[Math.floor(Math.random() * COLORS.length)]
    });
  }

  function drawPetal(p) {
    ctx.save();
    ctx.translate(p.x, p.y);
    ctx.rotate((p.rotation * Math.PI) / 180);
    ctx.globalAlpha = p.opacity;
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.ellipse(0, 0, p.size, p.size * 0.5, 0, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();
  }

  function animate() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    for (const p of particles) {
      p.y += p.speedY;
      p.x += p.speedX + Math.sin(p.y * 0.01) * 0.3;
      p.rotation += p.rotSpeed;

      // Wrap around when particle exits bottom
      if (p.y > canvas.height + 10) {
        p.y = -10;
        p.x = Math.random() * canvas.width;
      }

      drawPetal(p);
    }

    requestAnimationFrame(animate);
  }

  animate();
}
```

> **Important:** Call `initPetalCanvas()` inside `finishIntro()` (after `.unlocked`) or at the end of `script.js` — NOT on page load, to avoid rendering behind the locked intro layer.

### 3.3 — Scroll Reveal (IntersectionObserver)

Fade-and-slide elements into view as the user scrolls. Stagger siblings with CSS transition delays.

**CSS:**

```css
/* Base hidden state */
.reveal {
  opacity: 0;
  transform: translateY(30px);
  transition: opacity 0.8s ease, transform 0.8s ease;
}

.reveal.visible {
  opacity: 1;
  transform: translateY(0);
}

/* Staggered delays for sibling groups */
.reveal:nth-child(1) { transition-delay: 0s; }
.reveal:nth-child(2) { transition-delay: 0.12s; }
.reveal:nth-child(3) { transition-delay: 0.24s; }
.reveal:nth-child(4) { transition-delay: 0.36s; }
.reveal:nth-child(5) { transition-delay: 0.48s; }
.reveal:nth-child(6) { transition-delay: 0.60s; }
```

**JS:**

```js
function initScrollReveal() {
  const reveals = document.querySelectorAll('.reveal');
  if (!reveals.length) return;

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          entry.target.classList.add('visible');
          observer.unobserve(entry.target);     // Fire once only
        }
      });
    },
    {
      threshold: 0.15,
      rootMargin: '0px 0px -50px 0px'
    }
  );

  reveals.forEach(el => observer.observe(el));
}

// ╔════════════════════════════════════════════════════════════════╗
// ║  CRITICAL: initScrollReveal() is called ONLY inside          ║
// ║  finishIntro(), AFTER document.body.classList.add('unlocked') ║
// ║  NEVER call it at the top of script.js or on DOMContentLoaded║
// ╚════════════════════════════════════════════════════════════════╝
```

**Usage in HTML:** Add `class="reveal"` to any element that should animate in:

```html
<section class="section carousel-sec">
  <p class="section-subtitle reveal">LOOK WHO'S GROWING!</p>
  <h2 class="section-title reveal">Sweet Moments</h2>
  <div class="embla reveal">...</div>
</section>
```

---

## 4. Content Sections (Adaptable for Kid/Adult)

All sections share base classes (`.section`, `.section-title`, `.section-subtitle`) plus section-specific classes. Every section should be fully responsive with fluid typography using `clamp()`.

> **STRICT RULE:** Do NOT add any `max-height: 500px` restrictions, orientation locks, or landscape warning overlays. The design must flow naturally on ALL screen sizes and orientations.

### 4.1 — Hero Section

**HTML:**

```html
<section class="section hero-sec">
  <div class="heroTop">
    <p class="section-subtitle reveal">YOU'RE INVITED TO CELEBRATE</p>
    <!-- KID: "Baby's First Birthday!" / ADULT: "John's 50th Birthday" -->
    <h1 class="script-font reveal">Baby's First<br>Birthday!</h1>
  </div>

  <div class="heroBottom">
    <!-- KID: "A Little Honey is Turning One" / ADULT: "Half a Century of Greatness" -->
    <h2 class="reveal">A Little Honey is Turning One</h2>
    <p class="date reveal">Sunday, May 16th, 2027 | 10:00 AM</p>

    <!-- Countdown Timer -->
    <div class="cd-grid reveal">
      <div class="cd-box">
        <span class="cd-num" id="cd-m">0</span>
        <span class="cd-lbl">Months</span>
      </div>
      <div class="cd-box">
        <span class="cd-num" id="cd-w">0</span>
        <span class="cd-lbl">Weeks</span>
      </div>
      <div class="cd-box">
        <span class="cd-num" id="cd-d">0</span>
        <span class="cd-lbl">Days</span>
      </div>
      <div class="cd-box">
        <span class="cd-num" id="cd-h">0</span>
        <span class="cd-lbl">Hrs</span>
      </div>
      <div class="cd-box">
        <span class="cd-num" id="cd-s">0</span>
        <span class="cd-lbl">Secs</span>
      </div>
    </div>
  </div>

  <!-- Scroll Down Indicator (hidden until intro completes) -->
  <div class="scroll-down-indicator" id="scroll-indicator">
    <svg viewBox="0 0 24 24" fill="none" stroke-width="2"
         stroke-linecap="round" stroke-linejoin="round">
      <polyline points="7 13 12 18 17 13"></polyline>
      <polyline points="7 6 12 11 17 6"></polyline>
    </svg>
  </div>
</section>
```

**CSS (Hero):**

```css
.hero-sec {
  text-align: center;
  min-height: calc(var(--actual-vh, 1vh) * 100);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background: url('images/hero-banner-mobile.avif') center/cover no-repeat;
}

.hero-sec .script-font {
  font-size: clamp(3.5rem, 8vw, 6rem);
  color: var(--accent);
  line-height: 1.1;
  margin: clamp(20px, 4vw, 40px) 0;
  transform: rotate(-5deg);
}

.hero-sec h2 {
  font-size: clamp(1.5rem, 4vw, 2.5rem);
  color: var(--text);
}

.hero-sec .date {
  font-size: clamp(1.2rem, 3vw, 2rem);
  font-weight: 700;
  color: var(--text-light);
  margin-top: clamp(10px, 2vw, 20px);
}

/* Desktop: shift content to left, hero banner fills right */
@media (min-width: 768px) {
  .hero-sec {
    background-image: url('images/hero-banner-desktop.jpg');
    align-items: flex-start;
    text-align: left;
    padding: clamp(40px, 6vh, 80px) clamp(40px, 6vw, 100px);
  }
  .hero-sec .heroTop,
  .hero-sec .heroBottom {
    width: 100%;
    max-width: min(650px, 52vw);
    text-align: left;
  }
}
```

**JS (Countdown Timer):**

```js
/*
 * ==========================================
 * ⏳ CHANGE THE PARTY DATE & TIME HERE ⏳
 * ==========================================
 * Format: "Month Day, Year HH:MM:SS"
 * Example: "December 25, 2027 14:30:00"
 */
const targetDate = new Date('May 16, 2027 10:00:00').getTime();

function updateCountdown() {
  const now  = Date.now();
  const diff = targetDate - now;

  if (diff > 0) {
    const months = Math.floor(diff / (1000 * 60 * 60 * 24 * 30));
    const weeks  = Math.floor((diff % (1000 * 60 * 60 * 24 * 30)) / (1000 * 60 * 60 * 24 * 7));
    const days   = Math.floor((diff % (1000 * 60 * 60 * 24 * 7))  / (1000 * 60 * 60 * 24));
    const hours  = Math.floor((diff % (1000 * 60 * 60 * 24))      / (1000 * 60 * 60));
    const secs   = Math.floor((diff % (1000 * 60))                 / 1000);

    document.getElementById('cd-m').textContent = months;
    document.getElementById('cd-w').textContent = weeks;
    document.getElementById('cd-d').textContent = days;
    document.getElementById('cd-h').textContent = hours;
    document.getElementById('cd-s').textContent = secs;
  }
}

setInterval(updateCountdown, 1000);
updateCountdown();
```

**Countdown CSS:**

```css
.cd-grid {
  display: flex;
  justify-content: center;
  gap: clamp(8px, 1.5vw, 20px);
  margin-top: clamp(8px, 2vw, 24px);
  width: 100%;
}

.cd-box {
  flex: 1;
  padding: clamp(5px, 1.5vw, 15px);
  border-radius: 12px;
  background: var(--white);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.05);
  border: 2px solid var(--primary);
  text-align: center;
}

.cd-num {
  font-family: 'Fredoka One', cursive;
  font-size: clamp(16px, 4vw, 36px);
  display: block;
  color: var(--accent);
}

.cd-lbl {
  font-size: clamp(0.55rem, 1.5vw, 1.2rem);
  font-weight: 700;
  letter-spacing: 0.1em;
  color: var(--text-light);
  text-transform: uppercase;
}
```

**Scroll-Down Indicator CSS:**

```css
.scroll-down-indicator {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin-top: 8px;
  cursor: pointer;
  opacity: 0;
  transform: translateY(10px);
  transition: opacity 0.8s ease, transform 0.8s ease;
  pointer-events: none;
}

.scroll-down-indicator.visible {
  opacity: 1;
  transform: translateY(0);
  pointer-events: auto;
}

.scroll-down-indicator.hidden {
  opacity: 0 !important;
  pointer-events: none !important;
}

.scroll-down-indicator svg {
  width: 24px;
  height: 24px;
  stroke: var(--accent);
  animation: miniScrollBounce 1.5s infinite ease-in-out;
}

@keyframes miniScrollBounce {
  0%, 100% { transform: translateY(0); opacity: 0.7; }
  50%      { transform: translateY(8px); opacity: 1; }
}
```

**JS (hide on scroll):**

```js
window.addEventListener('scroll', () => {
  if (window.scrollY > 50) {
    document.getElementById('scroll-indicator')?.classList.add('hidden');
  }
});
```

### 4.2 — Gallery Section (Embla Carousel)

**HTML:**

```html
<section class="section carousel-sec">
  <p class="section-subtitle reveal">LOOK WHO'S GROWING!</p>
  <h2 class="section-title reveal">Sweet Moments</h2>

  <div class="embla reveal">
    <div class="embla__container">

      <!-- Slide 1 -->
      <div class="embla__slide">
        <div class="polaroid" style="--r: 1;">
          <!-- Optional decorative overlays (inline SVG or themed images) -->
          <img src="images/gallery-1.webp" alt="First Smiles"
               class="main-photo" loading="lazy">
          <span class="caption">First Smiles</span>
        </div>
      </div>

      <!-- Slide 2 -->
      <div class="embla__slide">
        <div class="polaroid" style="--r: 0;">
          <img src="images/gallery-2.webp" alt="Playtime Fun"
               class="main-photo" loading="lazy">
          <span class="caption">Playtime Fun</span>
        </div>
      </div>

      <!-- Add more slides as needed -->

    </div>
  </div>
</section>
```

**CSS:**

```css
.carousel-sec { background: var(--bg-accent); }

.embla {
  overflow: hidden;
  width: 100%;
  padding: 40px 0;
}

.embla__container {
  display: flex;
  touch-action: pan-y pinch-zoom;
  margin-left: -1rem;
}

.embla__slide {
  transform: translate3d(0, 0, 0);
  flex: 0 0 80%;
  min-width: 0;
  padding-left: 1rem;
}

.polaroid {
  background: #fff;
  padding: clamp(15px, 3vw, 40px) clamp(15px, 3vw, 40px) clamp(40px, 10vw, 100px);
  border-radius: 8px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
  position: relative;
  transform: rotate(calc(-3deg + (var(--r) * 6deg)));
}

.polaroid img.main-photo {
  width: 100%;
  height: clamp(300px, 45vw, 600px);
  object-fit: cover;
  border-radius: 4px;
  background: #eee;
}

.polaroid .caption {
  font-family: 'Pacifico', cursive;
  position: absolute;
  bottom: clamp(10px, 2vw, 20px);
  left: 0;
  width: 100%;
  text-align: center;
  font-size: clamp(1.2rem, 3.5vw, 2.8rem);
  color: #333;
}

/* Desktop: show 4 slides at once, no Embla swiping */
@media (min-width: 768px) {
  .carousel-sec .embla__container {
    display: flex;
    width: 100%;
    gap: clamp(12px, 1.8vw, 24px);
    margin-left: 0;
  }
  .carousel-sec .embla__slide {
    flex: 0 0 calc(25% - (clamp(12px, 1.8vw, 24px) * 3 / 4));
    padding-left: 0;
  }
  .carousel-sec .polaroid img.main-photo {
    height: clamp(200px, 20vw, 340px);
  }
}
```

**JS (Embla init):**

```js
const emblaNode = document.querySelector('.embla');
const emblaOptions = {
  loop: true,
  breakpoints: {
    '(min-width: 768px)': { active: false }   // Disable carousel on desktop
  }
};

const emblaApi = EmblaCarousel(emblaNode, emblaOptions, [
  EmblaCarouselAutoplay({ delay: 3000 })
]);
```

### 4.3 — Schedule Section

An image-based schedule with absolutely positioned text overlays, using container query units (`cqw`) for perfectly scaled typography:

**HTML:**

```html
<section class="schedule-sec">
  <div class="sticky-header">
    <p class="section-subtitle reveal">THE FUN AWAITS</p>
    <h2 class="section-title reveal">Party Schedule</h2>
  </div>

  <div class="schedule-image-wrapper">
    <img src="images/schedule-background.webp"
         alt="Schedule Background" class="schedule-bg-img" loading="lazy">

    <div class="events-overlay">
      <div class="event-board" style="top: 33%;">
        <p class="e-time">10:00 AM</p>
        <h3 class="e-title">Welcome & Play</h3>
      </div>
      <div class="event-board" style="top: 60%;">
        <p class="e-time">11:30 AM</p>
        <h3 class="e-title">Cake Cutting</h3>
      </div>
      <div class="event-board" style="top: 86%;">
        <p class="e-time">12:30 PM</p>
        <h3 class="e-title">Lunch & Goodbyes</h3>
      </div>
    </div>
  </div>
</section>
```

**CSS:**

```css
.schedule-sec {
  padding: 30px 0;
  background: var(--bg);
  position: relative;
  width: 100%;
  overflow: hidden;
}

.sticky-header {
  position: sticky;
  top: 0;
  z-index: 10;
  background: var(--bg);
  padding: 20px;
  text-align: center;
  border-bottom: 2px dashed var(--primary);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.03);
}

.schedule-image-wrapper {
  position: relative;
  width: 100%;
  container-type: inline-size;     /* Enables cqw units */
  display: block;
}

.schedule-bg-img {
  width: 100%;
  height: auto;
  display: block;
  object-fit: cover;
}

.events-overlay {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  z-index: 5;
}

.event-board {
  position: absolute;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 65%;
  text-align: center;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.event-board .e-time {
  font-weight: 700;
  color: #5c4331;
  font-size: 3.5cqw;
  margin: 0 0 1cqw 0;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.event-board .e-title {
  font-size: 5.5cqw;
  color: #3b2a1e;
  font-family: 'Fredoka One', cursive;
  margin: 0;
  line-height: 1.1;
}
```

### 4.4 — Gift Section (Kid Registry Modal / Adult "No Gifts" Note)

This section is **flexible by design**. Choose one of the two patterns:

#### Pattern A — Kid Theme: Gift Registry with Modal

**Section HTML:**

```html
<section class="section registry-sec">
  <p class="section-subtitle reveal">FOR THE BIRTHDAY STAR</p>
  <h2 class="section-title reveal">Book & Gift Raffle</h2>
  <img src="images/gift-illustration.webp" alt="Gift Illustration"
       class="registry-img reveal" loading="lazy">
  <p class="reveal" style="margin-bottom: 20px; font-weight: 500;">
    Bring a favorite storybook (signed with your sweet birthday wish)
    or a gift to be entered into our birthday raffle for a special prize!
  </p>
  <button id="open-registry-btn" class="sign-board reveal"
          style="border:none; cursor:pointer;">
    View Birthday Registry
  </button>
</section>
```

**Modal HTML (placed OUTSIDE `#app`):**

```html
<div id="registry-modal" class="modal-backdrop">
  <div class="modal-card">
    <button class="modal-close" id="modal-close-btn">&times;</button>
    <p class="modal-subtitle">GIFT & RAFFLE REGISTRY</p>
    <h3 class="modal-title">Pledge A Gift</h3>

    <div id="reg-warning" class="form-warning-box">
      Please enter your name and select a gift option!
    </div>

    <form id="registry-form">
      <label class="form-label">Your Full Name*</label>
      <input type="text" id="reg-name" class="input-field"
             placeholder="E.g., Jane Doe">
      <div class="invalid-feedback">Please enter your full name.</div>

      <label class="form-label">Select Gift Category*</label>
      <div class="registry-options" id="group-gift-cat">
        <label class="opt-card-radio">
          <input type="radio" name="gift_cat" value="Storybook">
          <div class="opt-card">
            <span class="opt-icon">📚</span>
            <div class="opt-info">
              <h4>Storybook</h4>
              <p>Children's book signed with a sweet wish inside</p>
            </div>
          </div>
        </label>
        <!-- Add more gift options as needed -->
      </div>
      <div class="invalid-feedback">Please select a gift category.</div>

      <label class="form-label">Gift Title / Item Details (Optional)</label>
      <input type="text" id="reg-note" class="input-field"
             placeholder="E.g., The Very Hungry Caterpillar book">

      <button type="submit" class="modal-btn">Pledge Gift via WhatsApp</button>
    </form>
  </div>
</div>
```

**Modal JS:**

```js
const regModal   = document.getElementById('registry-modal');
const openRegBtn = document.getElementById('open-registry-btn');
const closeRegBtn = document.getElementById('modal-close-btn');
const regForm    = document.getElementById('registry-form');
const regWarning = document.getElementById('reg-warning');

function openModal() {
  if (regModal) regModal.classList.add('active');
}
function closeModal() {
  if (regModal) {
    regModal.classList.remove('active');
    if (regWarning) regWarning.style.display = 'none';
    document.getElementById('reg-name')?.classList.remove('is-invalid');
    document.getElementById('group-gift-cat')?.classList.remove('is-invalid');
  }
}

if (openRegBtn) openRegBtn.addEventListener('click', (e) => { e.preventDefault(); openModal(); });
if (closeRegBtn) closeRegBtn.addEventListener('click', closeModal);
if (regModal) {
  regModal.addEventListener('click', (e) => {
    if (e.target === regModal) closeModal();   // Close on backdrop click
  });
}
```

#### Pattern B — Adult Theme: "Your Presence Is Present Enough"

```html
<section class="section registry-sec">
  <p class="section-subtitle reveal">A NOTE FROM US</p>
  <h2 class="section-title reveal">No Gifts Please</h2>
  <img src="images/champagne-toast.webp" alt="Champagne Toast"
       class="registry-img reveal" loading="lazy">
  <p class="reveal" style="margin-bottom: 20px; font-weight: 500; max-width: 500px; margin-inline: auto;">
    Your love and presence are the greatest gifts we could ask for.
    Please, no gifts — just bring yourself and your beautiful smile.
  </p>
</section>
```

### 4.5 — RSVP Section

(See full specification in §5 below.)

---

## 5. RSVP Form & WhatsApp Integration

### 5.1 — HTML Structure

```html
<section class="section rsvp-sec">
  <p class="section-subtitle reveal">WE HOPE TO SEE YOU</p>
  <h2 class="section-title reveal">Kindly RSVP</h2>

  <div class="form-card reveal">
    <form id="rsvp-form">

      <!-- Full Name (Required) -->
      <label class="form-label">Your Full Name*</label>
      <input type="text" id="r-name" class="input-field"
             placeholder="E.g., Jane Doe">
      <div class="invalid-feedback">Please enter your full name.</div>

      <!-- Phone Number (Required) -->
      <label class="form-label">Phone Number*</label>
      <input type="tel" id="r-phone" class="input-field"
             placeholder="E.g., +91 98765 43210">
      <div class="invalid-feedback">Please enter your phone number.</div>

      <!-- Attendance -->
      <label class="form-label">Will you be able to attend?*</label>
      <div class="radio-group" id="group-attendance">
        <label class="radio-btn">
          <input type="radio" name="attendance" value="Absolutely, wouldn't miss it!">
          <span class="radio-text">Absolutely, wouldn't miss it!</span>
        </label>
        <label class="radio-btn">
          <input type="radio" name="attendance" value="Sending love from afar.">
          <span class="radio-text">Sending love from afar.</span>
        </label>
      </div>
      <div class="invalid-feedback">Please select an attendance option.</div>

      <!-- Guest Toggle -->
      <label class="form-label">Will you be bringing a guest?*</label>
      <div class="toggle-group" id="group-has-guest">
        <label class="toggle-btn">
          <input type="radio" name="has_guest" value="Yes"
                 onchange="toggleGuestInput(true)">
          <span class="toggle-text">Yes</span>
        </label>
        <label class="toggle-btn">
          <input type="radio" name="has_guest" value="No"
                 onchange="toggleGuestInput(false)">
          <span class="toggle-text">No</span>
        </label>
      </div>
      <div class="invalid-feedback">Please select if you are bringing a guest.</div>

      <!-- Conditional Guest Name -->
      <div id="guest-name-wrap" style="display:none;">
        <label class="form-label">Guest Name(s)</label>
        <input type="text" id="r-guests" class="input-field"
               placeholder="Name(s) of accompanying guests">
      </div>

      <!-- Birthday Wish -->
      <label class="form-label">Leave a sweet birthday wish:</label>
      <textarea id="r-msg" class="input-field"
                placeholder="Your birthday wishes..."></textarea>

      <!-- Warning Box -->
      <div id="rsvp-warning" class="form-warning-box">
        Please fill out all required fields marked with *.
      </div>

      <button type="submit" class="submit-btn">Send RSVP via WhatsApp</button>
    </form>
  </div>
</section>
```

### 5.2 — Validation Logic

The JS must validate that **"Full Name"** and **"Phone Number"** are **not empty** before allowing submission. Show visual error styling on any missed field.

```js
function toggleGuestInput(show) {
  document.getElementById('guest-name-wrap').style.display = show ? 'block' : 'none';
}

const rsvpForm    = document.getElementById('rsvp-form');
const rsvpWarning = document.getElementById('rsvp-warning');

// Auto-clear validation errors on user input
document.querySelectorAll('#rsvp-form input, #rsvp-form textarea').forEach(input => {
  input.addEventListener('input', () => {
    input.classList.remove('is-invalid');
    const parentGroup = input.closest('.radio-group, .toggle-group');
    if (parentGroup) parentGroup.classList.remove('is-invalid');
  });
  input.addEventListener('change', () => {
    input.classList.remove('is-invalid');
    const parentGroup = input.closest('.radio-group, .toggle-group');
    if (parentGroup) parentGroup.classList.remove('is-invalid');
  });
});

rsvpForm.addEventListener('submit', function (e) {
  e.preventDefault();

  const nameEl       = document.getElementById('r-name');
  const phoneEl      = document.getElementById('r-phone');
  const name         = nameEl.value.trim();
  const phone        = phoneEl.value.trim();
  const attendanceEl = document.getElementById('group-attendance');
  const attendance   = document.querySelector('input[name="attendance"]:checked')?.value;
  const hasGuestEl   = document.getElementById('group-has-guest');
  const hasGuest     = document.querySelector('input[name="has_guest"]:checked')?.value;
  const guests       = document.getElementById('r-guests')?.value.trim() || '';
  const msg          = document.getElementById('r-msg')?.value.trim() || '';

  // Reset validation states
  nameEl.classList.remove('is-invalid');
  phoneEl.classList.remove('is-invalid');
  if (attendanceEl) attendanceEl.classList.remove('is-invalid');
  if (hasGuestEl)   hasGuestEl.classList.remove('is-invalid');
  if (rsvpWarning)  rsvpWarning.style.display = 'none';

  let isValid = true;

  // ── Required: Full Name ──
  if (!name) {
    nameEl.classList.add('is-invalid');
    isValid = false;
  }

  // ── Required: Phone Number ──
  if (!phone) {
    phoneEl.classList.add('is-invalid');
    isValid = false;
  }

  // ── Required: Attendance selection ──
  if (!attendance) {
    if (attendanceEl) attendanceEl.classList.add('is-invalid');
    isValid = false;
  }

  // ── Required: Guest toggle ──
  if (!hasGuest) {
    if (hasGuestEl) hasGuestEl.classList.add('is-invalid');
    isValid = false;
  }

  if (!isValid) {
    if (rsvpWarning) rsvpWarning.style.display = 'block';
    return;
  }

  // ── Build stylized WhatsApp message (see §5.3) ──
  const whatsappText = buildRSVPMessage({ name, phone, attendance, hasGuest, guests, msg });
  const encodedText  = encodeURIComponent(whatsappText);

  // ── Route to correct WhatsApp endpoint ──
  const hostNumber   = '918200482291';       // <-- CHANGE THIS to the host's number
  const whatsappUrl  = getWhatsAppUrl(hostNumber, encodedText);
  window.open(whatsappUrl, '_blank');

  // Reset form
  rsvpForm.reset();
  toggleGuestInput(false);
});
```

### 5.3 — WhatsApp Message Formatting (Luxury/Elegance Style)

The message uses bold/italic WhatsApp markdown, visual dividers, and contextual emojis for a premium feel:

```js
function buildRSVPMessage({ name, phone, attendance, hasGuest, guests, msg }) {
  const divider = '━━━━━━━━━━━━━━━━━━━━━━';

  let text = '';
  text += `${divider}\n`;
  text += `✨ *Baby's 1st Birthday — RSVP* ✨\n`;       // ← Adapt title per event
  text += `${divider}\n\n`;

  text += `👤 *Guest:* ${name}\n`;
  text += `📱 *Phone:* ${phone}\n`;
  text += `💌 *Response:* ${attendance}\n`;
  text += `👥 *Bringing Guest:* ${hasGuest}\n`;

  if (hasGuest === 'Yes' && guests) {
    text += `🧑‍🤝‍🧑 *Guest Name(s):* ${guests}\n`;
  }

  if (msg) {
    text += `\n🎂 *Birthday Wish:*\n_"${msg}"_\n`;
  }

  text += `\n${divider}\n`;
  text += `🎉 _Sent via Invite Box_\n`;
  text += `${divider}`;

  return text;
}
```

### 5.4 — Mobile vs. Desktop WhatsApp Routing

```js
function getWhatsAppUrl(hostNumber, encodedText) {
  // Detect mobile vs desktop
  const isMobile = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i
    .test(navigator.userAgent);

  if (isMobile) {
    // Mobile: deep-link to WhatsApp app
    return `https://api.whatsapp.com/send?phone=${hostNumber}&text=${encodedText}`;
  } else {
    // Desktop: open WhatsApp Web
    return `https://web.whatsapp.com/send?phone=${hostNumber}&text=${encodedText}`;
  }
}
```

### 5.5 — Gift Registry Modal WhatsApp Message (if using Pattern A)

```js
if (regForm) {
  regForm.addEventListener('submit', function (e) {
    e.preventDefault();

    const nameEl  = document.getElementById('reg-name');
    const name    = nameEl.value.trim();
    const giftCatEl = document.getElementById('group-gift-cat');
    const giftCat = document.querySelector('input[name="gift_cat"]:checked')?.value;
    const note    = document.getElementById('reg-note')?.value.trim() || '';

    nameEl.classList.remove('is-invalid');
    if (giftCatEl) giftCatEl.classList.remove('is-invalid');
    if (regWarning) regWarning.style.display = 'none';

    let isValid = true;
    if (!name)    { nameEl.classList.add('is-invalid'); isValid = false; }
    if (!giftCat) { if (giftCatEl) giftCatEl.classList.add('is-invalid'); isValid = false; }

    if (!isValid) {
      if (regWarning) regWarning.style.display = 'block';
      return;
    }

    const divider = '━━━━━━━━━━━━━━━━━━━━━━';
    let text = '';
    text += `${divider}\n`;
    text += `🎁 *Birthday Gift Raffle — Pledge* 🎁\n`;
    text += `${divider}\n\n`;
    text += `👤 *Guest Name:* ${name}\n`;
    text += `🎀 *Gift Category:* ${giftCat}\n`;
    if (note) {
      text += `📝 *Item / Note:* ${note}\n`;
    }
    text += `\n${divider}\n`;
    text += `🎉 _Sent via Invite Box_\n`;
    text += `${divider}`;

    const encodedText = encodeURIComponent(text);
    const hostNumber  = '918200482291';       // ← CHANGE THIS
    const whatsappUrl = getWhatsAppUrl(hostNumber, encodedText);
    window.open(whatsappUrl, '_blank');

    regForm.reset();
    closeModal();
  });
}
```

### 5.6 — Validation Error CSS

```css
.form-warning-box {
  background: #ffe6e6;
  border: 1px solid #ff4d4d;
  color: #cc0000;
  padding: 10px 14px;
  border-radius: 10px;
  font-size: 0.85rem;
  font-weight: 700;
  margin-bottom: 15px;
  display: none;
  text-align: center;
  animation: shakeWarning 0.4s ease-in-out;
}

@keyframes shakeWarning {
  0%, 100% { transform: translateX(0); }
  20%, 60% { transform: translateX(-5px); }
  40%, 80% { transform: translateX(5px); }
}

.invalid-feedback {
  display: none;
  color: #dc3545;
  font-size: 0.8rem;
  font-weight: 700;
  margin-top: 4px;
  margin-bottom: 12px;
  text-align: left;
}

.is-invalid + .invalid-feedback,
.is-invalid ~ .invalid-feedback {
  display: block;
}

.input-field.is-invalid {
  border-color: #dc3545 !important;
  background-color: #fff8f8 !important;
  box-shadow: 0 0 0 3px rgba(220, 53, 69, 0.15) !important;
}

.radio-group.is-invalid,
.toggle-group.is-invalid,
.registry-options.is-invalid {
  border: 2px solid #dc3545 !important;
  border-radius: 14px;
  padding: 6px;
  background-color: #fff8f8 !important;
}
```

---

## 6. Strict Footer Implementation

This EXACT footer HTML must be used. It handles iOS home indicators and Android gesture navigation bars safely using `env(safe-area-inset-*)`:

```html
<footer class="footer" style="padding: 8px 12px max(8px, env(safe-area-inset-left, 0px)); text-align: center; border-top: 1px solid rgba(231,191,133,0.2);">
  <p style="font-size:0.75rem; color:var(--text-muted);">
    Designed by <a href="https://invitebox.in/" target="_blank" rel="noreferrer" style="color:var(--primary); text-decoration:none; font-weight:600;">Invite&nbsp;Box</a>
  </p>
</footer>
```

**Do NOT modify this footer structure.** It is intentionally minimal and uses inline styles to guarantee correct safe-area spacing regardless of any external stylesheet cascade conflicts.

---

## 7. Build Checklist

**BUILD CHECKLIST (Must Verify All Before Completion):**

- [ ] Code is separated into `index.html`, `style.css`, and `script.js`.
- [ ] `viewport-fit=cover` included in meta tags.
- [ ] `setActualVh()` runs immediately as the first JS command.
- [ ] `history.scrollRestoration = 'manual'` and `scrollTo(0,0)` execute on load.
- [ ] `body` defaults to `overflow: hidden`; `.unlocked` is added only after intro.
- [ ] FAB and Scroll Indicator remain hidden until intro is complete and `reveal()` is called.
- [ ] Skip Intro button appears only after video starts playing and is hidden once intro finishes.
- [ ] WhatsApp message uses visual borders and bold/italic markdown.
- [ ] `initScrollReveal()` called ONLY after `.unlocked` is added.
- [ ] All icons are inline SVGs.
- [ ] Fully responsive on mobile and desktop (NO landscape warnings or max-height locks).

---

> **Document version:** 1.0  
> **Last updated:** November 2026  
> **Author:** Invite Box Engineering

