# E-Invites For Weddings: Common Requirements & Design Specifications

This document outlines the standard architecture, common components, and design guidelines extracted from analyzing all invitation templates (`GaneshaEntry.html`, `DoorDesign.html`, `NewInvitaionOne.html`, `index.html`, `AnjaliManeetInvitation.html`, `curtainDesign.html`, `curtainDesign2.html`).

All future wedding invitation designs and projects **must** strictly adhere to these specifications to guarantee consistency, high performance, and premium user experience.

---

## 1. Core Architecture & Layout Rules

### 1.1 Mobile-First Centered Container
- **Max Width**: The main invitation container (`#app` or `#main`) must have a max-width of `480px` centered on the viewport (`margin: 0 auto; min-height: 100vh`).
- **Outer Viewport Background**: Dark background (`#1a1a1a` or `#111111`) outside the 480px container to focus user attention on mobile layout.

### 1.2 Portrait Orientation Lock Guard
- **Landscape Warning Screen**: Every page includes a fixed `#landscape-warning` element that is displayed via CSS media query when the device is in landscape mode on small devices:
  ```css
  @media (orientation: landscape) and (max-height: 500px) {
    #landscape-warning { display: flex !important; }
  }
  ```
- Prompts the guest: *"Please Rotate Your Device — This invitation is beautifully crafted for portrait viewing."*

---

## 2. Opening Animation & Scroll Lock Sync

### 2.1 Page Scroll Lock
- **Initial State**: Scrolling is locked on load (`body { overflow: hidden; }`) while the entry sequence (Video overlay, 3D Door opening, Curtain sliding, or Envelope opening) plays.
- **Scroll Restoration**: Force manual scroll restoration to always start at top `(0,0)` on page load or reload:
  ```js
  if ('scrollRestoration' in history) history.scrollRestoration = 'manual';
  window.scrollTo(0, 0);
  ```

### 2.2 Unlock Triggering
- Scrolling **must only be unlocked** (`document.body.classList.add('unlocked')`) at the exact moment the entry animation completes:
  - Video ended event / fade-out
  - Door open transform complete
  - Curtain slide complete
  - Envelope open animation finish

---

## 3. Continuous Scroll-Down Indicator

### 3.1 Small & Elegant Design
- **Icon Sizing**: Small, subtle 18px double-chevron inline SVG icon (`stroke-width: 2.5`).
- **Typography**: Compact label (`font-size: 0.6rem; letter-spacing: 1.5px; text-transform: uppercase;`).
- **Color**: Matches the page's primary theme color (`var(--color-primary)` or `var(--gold)`).

### 3.2 Animation & Interactivity
- **Continuous Bounce**: Smooth keyframe bounce animation:
  ```css
  @keyframes miniScrollBounce {
    0%, 100% { transform: translateY(0); opacity: 0.7; }
    50% { transform: translateY(6px); opacity: 1; }
  }
  ```
- **Unlock Visibility**: Hidden during intro animation (`opacity: 0; pointer-events: none;`). Added `.visible` class only when scrolling is unlocked (`body.unlocked`).
- **Smooth Scroll Click**: Tapping indicator smoothly scrolls to the next section (`element.scrollIntoView({ behavior: 'smooth' })`).
- **Auto-Hide on Scroll**: Fades out automatically when user scrolls past 50px (`window.scrollY > 50`).

---

## 4. Floating Music Control (BGM On/Off)

### 4.1 Audio Controller & Autoplay Policy
- Background music element: `<audio id="bg-music" loop src="..."></audio>`.
- **Autoplay Handling**: Attempts `bgMusic.play()` on DOM load with low initial volume (`0.15`). Listens for first user interaction (`click` / `touchstart`) as fallback for browser autoplay restrictions.

### 4.2 Floating Music FAB (`#mfab`)
- Positioned fixed at `bottom: 25px; right: 25px;` inside or relative to the mobile container width.
- Displays inline SVG muted / unmuted speaker icons depending on audio state.
- **Zero Icon Font Dependency**: Toggle relies strictly on inline `<svg>` elements or native SVGs (never FontAwesome or icon webfonts).

---

## 5. WhatsApp RSVP Form Integration

### 5.1 Interactive Input Fields
Every invitation template includes an interactive RSVP section:
- **Guest Name Input**: Text field for full name.
- **Team Cheer Selector**: Side selection toggle (`Team Bride` vs `Team Groom`).
- **Attendance Toggle**: Buttons for `Joyfully Attending` vs `Regretfully Declining`.
- **Guest Count Selector**: Dropdown / stepper for number of guests attending (1, 2, 3, 4+).
- **Event Checklist**: Interactive checkboxes for attending specific functions (Mehndi, Sangeet, Haldi, Wedding, Reception).

### 5.2 Direct WhatsApp Message Dispatch
Form submission compiles formatted markdown text and redirects to WhatsApp Web / App:
```js
const whatsappNumber = "918200482291"; // Host phone number with country code
const encodedMessage = encodeURIComponent(message);
window.open(`https://wa.me/${whatsappNumber}?text=${encodedMessage}`, '_blank');
```

---

## 6. Footer & Branding Standard

### 6.1 Traditional Blessings & Names
- Om / Religious ornament symbol (`ॐ` or `✦ ✦ ✦`).
- Blessing phrase: *"With the blessings of our elders and the warmth of your presence."*
- Couple names in script or serif font.

### 6.2 Mandatory Credit Line
Every invitation footer **must** conclude with the official Invite Box credit link:
```html
<p class="made-with-love">Designed by <a href="https://invitebox.in/" target="_blank" rel="noreferrer">Invite&nbsp;Box</a></p>
```

---

## 7. Performance Guidelines: Inline SVG & Asset Optimization

### 7.1 Strict Requirement: Inline SVG Code (No Icon Libraries)
- **NO FontAwesome, Bootstrap Icons, Material Icons, or external icon fonts**.
- All icons (music notes, speaker icons, chevrons, WhatsApp logo, location pins, calendars, Ganesha icons, dividers) **must be written directly as inline `<svg>` elements**.
- **Reasoning**: Eliminates external HTTP requests, eliminates render-blocking CSS, prevents icon flash/layout shifts, and speeds up page load on mobile networks.

### 7.2 Image & Video Optimization
- WebP format for all static background images, photo frames, and scratch card masks (`.webp`).
- H.264/AAC MP4 format for video overlays with `playsinline muted disablePictureInPicture` attributes.

---

## 8. Photo Carousels (Embla Carousel Recommended)

### 8.1 Modern Touch Carousel Standard
- When photo galleries, couple memories, or timeline carousels are required in a design, use **Embla Carousel** (or lightweight Swiper when pre-integrated with specific effects).
- Must support touch swiping, auto-play slide transitions, and optional lightbox/Fancybox zoom preview on click.
- Auto-play must be paused/restarted cleanly when elements enter/leave viewport.

---

## 9. Common Interactive Extras Matrix

| Feature | Description | Implementation Standard |
| :--- | :--- | :--- |
| **Falling Petals Canvas** | Particle animation of swaying flower petals | HTML5 Canvas (`#petals-canvas`) with requestAnimationFrame |
| **Countdown Timer** | Real-time countdown (Days, Hours, Mins, Secs) | `setInterval` targeting wedding timestamp |
| **Scratch Card** | Interactive gold/silver reveal overlay | HTML5 Canvas scratch effect with mouse/touch move events |
| **Lightbox Gallery** | High-res full-screen image gallery | Fancybox `[data-fancybox]` integration |
| **AOS Scroll Reveal** | Fade/slide animations on section scroll | IntersectionObserver or AOS library |

---

## Summary Checklist for Future Invitation Designs

- [ ] Mobile-first container set to max-width `480px` with centered layout.
- [ ] Portrait landscape warning overlay active.
- [ ] Body scroll locked (`overflow: hidden`) until intro animation finishes.
- [ ] Small bouncing scroll-down arrow (`.scroll-down-indicator`) revealed upon scroll unlock.
- [ ] Auto-playing background music with floating SVG toggle button (`#mfab`).
- [ ] Interactive RSVP form connecting to host WhatsApp API.
- [ ] Standard footer with `Designed by Invite Box` link (`https://invitebox.in/`).
- [ ] **100% Inline SVG icons** (zero FontAwesome / icon webfont libraries).
- [ ] Embla Carousel for photo galleries with auto-play capabilities.