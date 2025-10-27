# WebGL Host Template — Fullscreen / PWA / Multi‑platform Spec

## Purpose
A concise, developer‑facing specification for a single `index.html` + assets template that hosts a Unity WebGL build and provides optimized UX for:
- Desktop / Chromebook (wide screens)
- Android (Chrome: supports true fullscreen + orientation lock)
- Apple devices (Safari tab: in‑page with instructions)
- Apple devices launched via **Add to Home Screen** (PWA standalone: true fullscreen, landscape)

This document provides the file layout, code snippets, platform detection logic, CSS layout patterns, and a test/acceptance checklist. Hand this to a front‑end dev and they should be able to implement a working template matching the described behavior.

---

## Goals and requirements
1. Single page (`index.html`) that shows:
   - Page title at top
   - Unity canvas (interactive) prominently
   - Supplemental content (description, links, contact) below the canvas — scrollable
2. Platform‑specific UX:
   - **Desktop / Chromebook**: two‑column or single column layout; fullscreen button available and uses `requestFullscreen()`.
   - **Android (Chrome)**: same as desktop but fullscreen button triggers `requestFullscreen()` + `screen.orientation.lock('landscape')` so the screen rotates to landscape like YouTube.
   - **iOS Safari (browser tab)**: fullscreen API/orientation lock are effectively blocked; show a small instruction banner explaining "Add to Home Screen for best experience" and a short how‑to. Allow playing in‑page but the layout should avoid tall/portrait UI by limiting height.
   - **iOS PWA (Add to Home Screen / standalone)**: runs as a standalone app; no browser chrome; should open fullscreen and prefer landscape orientation. Hide the supplemental content if desired (or keep it behind a scrollable view).
3. Resizable canvas behavior: never become taller than a defined aspect threshold (e.g. height <= width / 1.4) — allow wider aspect ratios.
4. Fullscreen should be requested **only from the page JS** (not via `unityInstance.SetFullscreen(1)`), in order to maintain a single user gesture and to call `screen.orientation.lock()` afterwards.
5. Clean fallbacks for environments that don't support orientation lock.

---

## Files and assets (recommended layout)
```
/ (site root)
  index.html                  <-- main hosting page (single source of truth)
  manifest.json               <-- PWA manifest (display: standalone, orientation: landscape)
  /Build/                     <-- Unity WebGL build output
    V1.0_B1.data
    V1.0_B1.framework.js
    V1.0_B1.loader.js
    V1.0_B1.wasm
  /TemplateData/
    style.css                 <-- Unity build CSS — we will override specific rules here
    favicon.ico
  /assets/
    icons/                    <-- apple touch icon, manifest icons
```

---

## High level page structure (HTML)
- `#page-title` — visible on all platforms
- `#unity-section` — contains the Unity canvas, loading bar, footer (fullscreen button), controls
- `#site-content` — article / description / contact / links (scrolls under the canvas)
- platform specific banners: `#ios-hint`, `#pwa-hint`

**Example (skeleton)** — put this inside `index.html` body (implementor will integrate into existing Unity template):

```html
<header id="page-title">Stem Cell Zoo</header>
<main>
  <section id="unity-section">
    <!-- Unity container (existing Unity code) -->
    <div id="unity-container">
      <canvas id="unity-canvas"></canvas>
      <div id="unity-loading-bar">...</div>
      <div id="unity-footer">
        <div id="unity-fullscreen-button" role="button" aria-label="Fullscreen"></div>
      </div>
    </div>
  </section>

  <section id="site-content">
    <article id="description"> <!-- placeholder content -->
      <h2>About this activity</h2>
      <p><!-- developer adds text / images / links here --></p>
    </article>
  </section>
</main>

<!-- Helpful UX banners -->
<div id="ios-hint" class="platform-hint" hidden>
  <p>For the best experience on iPad, tap Share → Add to Home Screen. Then reopen the game from your home screen.</p>
</div>

<div id="pwa-hint" class="platform-hint" hidden>
  <p>Running as an app. Rotate device to landscape.</p>
</div>
```

---

## Key meta tags & manifest (head)
Add in `<head>`:
```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<link rel="manifest" href="/manifest.json">
<!-- iOS web app config -->
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="Stem Cell Zoo">
<link rel="apple-touch-icon" href="/TemplateData/favicon.ico">
```

**manifest.json** (example):
```json
{
  "name": "Stem Cell Zoo",
  "short_name": "StemCell",
  "display": "standalone",
  "orientation": "landscape",
  "start_url": "./index.html",
  "icons": [
    { "src": "assets/icons/icon-192.png", "sizes": "192x192", "type": "image/png" }
  ]
}
```

> Note: iOS ignores `display`/`orientation` in some versions but the manifest is still useful for Android/Chrome and modern browsers.

---

## Platform detection logic (JS)
Place this script before the Unity loader script so the page can adjust layout before Unity renders.

```js
const ua = navigator.userAgent || '';
const isIOS = /iPhone|iPad|iPod/.test(ua);
const isAndroid = /Android/.test(ua);
const isStandalone = window.matchMedia('(display-mode: standalone)').matches || window.navigator.standalone === true;

// css class toggles for layout
if (isStandalone) document.documentElement.classList.add('pwa');
if (isIOS) document.documentElement.classList.add('ios');
if (isAndroid) document.documentElement.classList.add('android');
if (!isIOS && !isAndroid) document.documentElement.classList.add('desktop');

// show/hide hints
if (isIOS && !isStandalone) document.getElementById('ios-hint').hidden = false;
if (isStandalone) document.getElementById('pwa-hint').hidden = false;
```

---

## Canvas resize policy (JS + CSS)
Goal: canvas should **fill width** and use `height = min(window.innerHeight, window.innerWidth / maxAspect)` so it never gets taller than `width / maxAspect`.

- `maxAspect = 1.4` (you mentioned `width / 1.4` earlier). This means height ≤ width / 1.4.

**CSS (base)**
```css
html, body { height:100%; margin:0; }
#unity-section { display:flex; justify-content:center; align-items:center; background:#000; }
#unity-container { width:100%; max-width:100vw; }
#unity-canvas { display:block; width:100%; height:auto; }
```

**JS (resize helper)**
```js
function resizeUnityCanvas(maxAspect = 1.4) {
  const canvas = document.getElementById('unity-canvas');
  const w = window.innerWidth;
  const h = window.innerHeight;
  const allowedHeight = Math.min(h, Math.floor(w / maxAspect));
  canvas.style.width = w + 'px';
  canvas.style.height = allowedHeight + 'px';
}
window.addEventListener('resize', () => resizeUnityCanvas(1.4));
resizeUnityCanvas(1.4);
```

When orientation lock is *not supported* (iOS browser), this limits the vertical size so the UI never becomes a tall portrait layout.

---

## Fullscreen + orientation handler (replace `unityInstance.SetFullscreen(1)` usage)
**Important**: call the browser Fullscreen API from page JS (user gesture) and then call `screen.orientation.lock('landscape')` (if available). Do not call `unityInstance.SetFullscreen(1)` — it makes Unity attempt fullscreen internally and can produce permission errors.

**Add this helper and wire it to the fullscreen button**:

```js
async function enterFullscreenLandscape(containerSelector = 'html') {
  const el = document.querySelector(containerSelector) || document.documentElement;
  try {
    // Request fullscreen
    if (el.requestFullscreen) await el.requestFullscreen();
    else if (el.webkitRequestFullscreen) await el.webkitRequestFullscreen(); // Safari

    // lock orientation (works on Android Chrome, not on iOS Safari tab)
    if (screen.orientation && screen.orientation.lock) {
      await screen.orientation.lock('landscape').catch(() => {});
    }

    // notify Unity if needed (optional)
    if (window.unityInstance && typeof window.unityInstance.SendMessage === 'function') {
      // GameObject name and method are project-specific; create a handler in Unity if required.
      window.unityInstance.SendMessage('GameController', 'OnEnterFullscreen');
    }
  } catch (err) {
    console.log('Fullscreen/orientation failed or blocked:', err);
  }
}

// wire after unityInstance is created
fullscreenButton.onclick = () => enterFullscreenLandscape('#unity-container');
```

**Notes**:
- `screen.orientation.lock()` is only allowed after a user gesture and usually after entering fullscreen. That’s why it’s done *after* `requestFullscreen()`.
- If the browser denies it the code should gracefully continue: use the resize helper as a fallback to keep the canvas landscape‑like.

---

## iOS specifics (Safari tab vs. Add to Home Screen)
- **Safari tab**: `screen.orientation.lock` is not supported, and the browser will show address and toolbars. Provide an on‑screen instruction (banner) showing steps to add the page to Home Screen and recommend launching from Home Screen for best experience.
- **PWA / Add to Home Screen**: when launched from Home Screen, the site runs in `display-mode: standalone`. You can detect this via `window.matchMedia('(display-mode: standalone)').matches` or `navigator.standalone`. In standalone mode:
  - The page has no browser chrome.
  - You can present a landscape‑first layout and (with modern iOS versions) orientation locking can work better.
  - Hide the `#ios-hint` banner when `isStandalone`.

Provide a short explainer element in `#ios-hint` with a small image showing the Share → Add to Home Screen steps.

---

## Layout variations (desktop vs mobile stacking)
Use CSS media queries plus the `desktop/android/ios/pwa` body classes added earlier.

**Desktop / Chromebook** (two column suggestion)
```css
@media (min-width: 900px) {
  main { display:grid; grid-template-columns: 1fr 360px; gap:20px; }
  #unity-section { grid-column: 1 / 2; }
  #site-content { grid-column: 2 / 3; }
}
```

**Mobile / stacked**
```css
@media (max-width: 899px) {
  main { display:block; }
  #unity-section { width:100%; }
  #site-content { width:100%; }
}
```

If `documentElement` has class `pwa`, you may hide `#site-content` or collapse it into a menu to keep the app focused on gameplay.

---

## UX behaviors & small details
- **Fullscreen button** always visible on Desktop & Android. On iOS Safari tab, show a small `Open in Home Screen` CTA instead.
- **Rotate hint**: when in fullscreen and the width < height (i.e., portrait), show an overlay with a rotate animation/message until the device is rotated.
- **Touch targets**: ensure fullscreen and important UI have >=44px tappable area on mobile.
- **Keyboard input**: WebGL input fields on mobile can cause the viewport to shrink — test acceptance flows where keyboard is used.

---

## Unity integration notes
- Set `config.matchWebGLToCanvasSize` in the Unity loader if you plan on controlling canvas size from JS. In the config block you can set `config.matchWebGLToCanvasSize = false;` and manually set the canvas resolution to reduce GPU load. Otherwise Unity will resize the internal render target to the canvas DOM size automatically.
- Don’t call `unityInstance.SetFullscreen(1)` from JavaScript — use the full browser Fullscreen API as shown above.
- Use a Unity GameObject named `GameController` with methods `OnEnterFullscreen` / `OnExitFullscreen` if you need to adjust UI in code when fullscreen changes.

---

## Testing checklist
**Local dev (VS Code Live Server)**
1. Run Live Server and verify local page loads: `http://127.0.0.1:5500/index.html` and `http://<LAN-IP>:5500/index.html` from a phone on the same Wi‑Fi.
2. Test Desktop: click fullscreen button → page fills (address bar hidden on desktop).
3. Test Chromebook: click fullscreen → page enters fullscreen; call `screen.orientation.lock('landscape')` should succeed.
4. Test Android (Chrome): click fullscreen → page enters fullscreen and rotates to landscape.
5. Test iOS Safari tab: visit page → ios hint banner appears; clicking fullscreen should degrade gracefully (likely blocked). Test Add to Home Screen: add and relaunch → page runs standalone and should occupy full screen without chrome.

**Acceptance criteria**
- Desktop/Chromebook: game canvas visible, fullscreen uses full screen, supplemental content readable below/aside, allowed to scroll past the canvas.
- Android: fullscreen button rotates to landscape and hides browser chrome.
- iOS tab: hint & instructions visible; launch from Home Screen results in fullscreen standalone app with landscape prompt.
- Canvas never exceeds defined `maxAspect` height limit to avoid tall portrait UI.

---

## Implementation notes and gotchas
- Some older iOS versions ignore orientation lock even in standalone mode. Provide fallbacks and messaging.
- `screen.orientation.lock()` must be called after a user gesture and often after `requestFullscreen()` returns a resolved promise.
- For a truly app‑like distribution (App Store), wrapping the WebGL build in a WebView is possible, but requires app signing and store submission; PWA is simpler and free for classrooms.
- Consider adding a small service worker + caching strategy if you want offline availability in PWA mode.

---

## Deliverables for handoff
1. `index.html` built from the skeleton above with Unity loader integration and three content sections (desktop / android / ios-hint).  
2. `manifest.json` and icons.  
3. `TemplateData/style.css` edits or an overriding `<style>` block to force show/hide of footer and fullscreen button on mobile.  
4. `screen-resize.js` (optional): a small file with the `resizeUnityCanvas()` and `enterFullscreenLandscape()` helpers.  
5. QA checklist and sample test device list (Chromebook, Android phone, iPad on current iOS version).

---

## Quick code location summary (changes to make in existing project)
- `index.html`:
  - Add recommended `<meta>` tags + `<link rel="manifest">` in `<head>`.
  - Add platform detection script before the Unity loader script.
  - Replace `unityInstance.SetFullscreen(1)` call with `enterFullscreenLandscape()` wiring shown above.
  - Add banners `#ios-hint`, `#pwa-hint` and `#site-content` (article placeholder).
  - Add `resizeUnityCanvas()` invocation and `window.resize` listener.
- `TemplateData/style.css`:
  - Comment out or remove `.unity-mobile #unity-footer { display: none; }` so fullscreen button is visible.
  - Add responsive rules for layout (two column at large widths, stacked at small widths).
- `manifest.json` — add to root and reference from head.