# Kitchen Studio — Deployment options

How to run [index.html](index.html) on different devices. The app is a single self-contained
HTML file (see [SPEC.md](SPEC.md)); these notes cover **where to put it** so it runs well —
especially on an **iPad**.

---

## Key facts before you start

- **Serve it over http(s) — don't open the raw file on a tablet/phone.** iOS Safari blocks
  ES-module / importmap scripts loaded from `file://` (origin `null`, CORS), so the app won't
  start if you just open the file from the Files app. Desktops can still double-click the file.
- **First load needs internet.** Three.js is pulled from a CDN (unpkg) on first load, then the
  browser caches it. After that it works offline *until the browser evicts the cache*.
- **Everything is local & private.** No accounts, no server logic. Designs autosave to the
  browser's `localStorage` on that device; use **Export JSON** to move a design between devices.

---

## Desktop / laptop (Windows, macOS, Linux)

- **Double-click [index.html](index.html)** — opens in the default browser and runs from `file://`.
  This is the original design goal and works on desktop browsers.
- Or serve it (see below) and open the URL — identical result, and required if you later add a
  service worker.

---

## iPad / iPhone

### Option A — Publish it (recommended; nothing left running on the PC)

1. On the PC, open **app.netlify.com/drop** (or set up **GitHub Pages**) and **drag
   `index.html`** onto the page. You get a URL such as `https://yourname.netlify.app`.
2. On the iPad, open that URL in **Safari**.
3. Tap **Share → Add to Home Screen** for a full-screen, app-like icon (no browser chrome).

- **Pros:** works anywhere, easy to re-share, update by re-dragging the file.
- **Cons:** the design is public at that URL unless you password-protect the host.

### Option B — Serve from your PC over Wi-Fi (quick, private)

Both devices on the **same network**:

1. On the PC, in the project folder run **one** of:
   ```powershell
   python -m http.server 8000
   # or, if you have Node:
   npx serve -l 8000
   ```
2. Find the PC's IPv4 address:
   ```powershell
   ipconfig    # look for "IPv4 Address", e.g. 192.168.1.42
   ```
3. On the iPad, open Safari at `http://192.168.1.42:8000/index.html`.

- **Pros:** instant, private, no publishing. **Cons:** the PC must stay on and serving.

### Option C — Local-server iOS app (no PC, no publishing)

Use an app that serves files locally over `http://localhost`, e.g. **WorldWideWeb** (The
Iconfactory) or **Documents by Readdle**. Put `index.html` in the app and open it through its
built-in local server (not as a `file://`). This makes the ES modules load correctly.

### Not recommended — opening the file directly

AirDropping / iCloud-syncing the file and tapping it in **Files** opens it via `file://`, where
the module imports are blocked, so the 3D view won't load. Use A, B or C instead.

---

## Using it by touch (iPad)

- **Orbit** = one-finger drag · **Zoom** = pinch · **Pan** = two-finger drag
  (Three.js OrbitControls handle touch automatically).
- The Colours list's live **hover-preview** doesn't fire on touch, but **tapping a colour still
  applies** it.
- **Plan → Print / PDF:** use the Print button, or Safari **Share → Print → pinch the preview to
  open the PDF → Save to Files**. The A4-landscape print layout still applies.
- **PNG / Snapshot** downloads land in **Files → Downloads**.

---

## Moving a design between devices

Designs are stored per-device in `localStorage`. To transfer one:

1. On the source device: **Design tab → Export JSON** (`kitchen-design.json`).
2. Send it across (email, AirDrop, iCloud, etc.).
3. On the target device: **Design tab → Import JSON**.

---

## Optional enhancements (not yet implemented)

Ask if you'd like any of these added:

1. **Installable, offline PWA** — web-app manifest + Apple touch meta tags + a service worker so
   it works **fully offline** after first load. Optionally **inline Three.js** so it never needs
   the CDN at all (fully self-contained).
2. **iPad-friendly layout** — collapsible / bottom-sheet panel in portrait, larger tap targets,
   and `viewport-fit=cover` plus disabling accidental page pinch-zoom so only the 3D view zooms.
3. **Password-protected hosting** — e.g. a Netlify/Cloudflare Access rule if the published URL
   shouldn't be public.
