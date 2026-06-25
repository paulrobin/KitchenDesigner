# Kitchen Studio — Project Specification

A single-file, browser-based 3D kitchen designer for planning a real kitchen install and
auditioning **Farrow & Ball** colours on it. Built to be opened by double-clicking one
HTML file — no install, no build step, works offline.

- **Deliverable:** [index.html](index.html) (one self-contained file)
- **Status:** Working prototype, actively evolving
- **Current version:** v3 (parametric / spec-driven)
- **Owner:** personal project (kitchen renovation planning)

---

## 1. Purpose & goals

Help decide on a new kitchen by:

1. Building a 3D model that can be made to **closely match a real kitchen** (layout, units, appliances, window).
2. **Auditioning colours live** on cabinets, walls, worktop, floor and handles using the real Farrow & Ball range.
3. Saving, comparing and exporting designs so nothing is lost.

### Non-goals (deliberately out of scope)
- Not a CAD / manufacturing tool (no joinery cut lists, no exact bespoke profiles).
- Not photoreal to paint-accuracy — F&B hexes are on-screen approximations (see §9).
- No accounts, no cloud, no server. Local-only.

---

## 2. Key constraints & decisions (with rationale)

| Decision | Rationale |
|---|---|
| **Single HTML file**, opened by double-click | Zero-install, easy to share, runs from `file://`. The original hard requirement. |
| **No local JS modules / no bundler** | ES module `import './x.js'` is blocked under `file://` (CORS, origin `null`). Keeping all our code inline + importing Three.js from an absolute CDN URL preserves double-click. |
| **Three.js from CDN** (unpkg) via importmap | No npm/build. Needs internet on **first load** only (browser caches it). |
| **Procedural textures & materials only** (no image files) | Keeps it offline/self-contained and keeps the canvas untainted so **snapshots** work. |
| **F&B colours: names/numbers exact, hex approximate** | F&B do not publish official RGB; their website swatches are images. Hexes are close visual matches; users cross-check against real sample cards (name + No. shown everywhere). |
| **Data-driven model** (`spec` object → `buildKitchen`) | So the kitchen can be reconfigured from the UI and saved, instead of being hardcoded. |

### When to "graduate" to a real project (Vite)
Move off single-file **only** if we add: multiple saved room designs, an image-texture/asset
library, a proper component/appliance catalogue, or want to **host it online**. At that point use
Vite (one-command dev server + a build that can still output a single shareable file).

---

## 3. How to run / develop

- **Run:** double-click [index.html](index.html) (Chromium/Edge/Firefox, reasonably modern). First load needs internet for Three.js.
- **Edit:** everything is in [index.html](index.html) — `<style>`, the tabbed `<aside id="panel">`, and one `<script type="module">`.
- **Validate:** check for syntax errors after edits; reload the browser to test (no hot reload).
- **Performance escape hatch:** the **Photoreal** toggle disables post-processing (GTAO/bloom) on weaker GPUs; the post-FX pipeline also auto-falls back to plain rendering if it can't initialise.

---

## 4. Architecture overview

```
index.html
├── <style>            UI styling (dark panel, tabs, form controls, colour picker)
├── <aside id="panel"> Tabbed UI: Design | Colours | Scene
└── <script type=module>
    ├── maths + procedural noise (fbm) and texture builders
    ├── PALETTE (301 F&B colours) + SURFACES + DEFAULT_COLOURS
    ├── renderer / scene / camera / OrbitControls / RoomEnvironment IBL
    ├── lights: HemisphereLight + DirectionalLight "sun" + RectAreaLight (window) + Sky
    ├── applyTimeOfDay(t)  — drives sun arc/colour, sky, exposure, window light, LED
    ├── procedural PBR textures (wood, stone, plaster, paint, brushed metal)
    ├── materials (shared, one per recolourable role + fixed materials)
    ├── geometry helpers: rbox, flatbox, barHandle, shakerFace
    ├── makeUnit / makeWallCab / makeSinkFixtures / makeHob / makeExtractor
    ├── SPEC model + DEFAULT_SPEC + load/save (localStorage)
    ├── buildKitchen(spec): frames → walls → runs → window/doorway → island → garden → props
    ├── EffectComposer post-FX (GTAO + bloom + SMAA) with try/catch fallback
    ├── colour system: per-surface dropdown picker (search + live hover-preview)
    ├── presets + compare (A/B)
    ├── Design pane renderer (spec form) + JSON export/import
    └── boot + render loop
```

**Build/rebuild model:** `buildKitchen(spec)` populates a single `kitchenGroup`; on any spec
change `rebuildKitchen()` disposes its geometry and rebuilds. Lights, sky, environment and
camera live **outside** `kitchenGroup` so they persist across rebuilds.

### Coordinate system / frames
- Corner of the kitchen at the world origin `(0,0)`. Floor at `y≈0`; up is `+y`.
- **Run A** = back wall (faces `+z`, rotation 0). **Run B** = left wall (faces `+x`, +90°). **Run C** = right wall (faces `-x`, −90°).
- Runs B and C start at offset `BASE_DEPTH` because Run A occupies the corner.
- Units are built facing `+z`, then rotated by the run's `faceRot` and positioned via
  `framePoint(frame, along, across)`. `placeBox` derives world X/Z sizes from the frame's
  direction/normal so the same code works on any wall.
- `roomW = max(run A length, 1.8)`, `roomD = max(BASE_DEPTH + run B/C length, 2.6)`.

---

## 5. Data model

### 5.1 `spec` (the kitchen design)
```js
spec = {
  layout: 'single' | 'L' | 'U',
  ceiling: 2.6,                       // metres
  worktopEndOverhang: 0.01,           // metres - worktop overhang at exposed run ends (default 10mm)
  handleWidth: 0.13,                  // metres - global bar-handle length (default 130mm)
  endPanels: true,                    // 15mm finished panels on exposed run ends
  runs: {
    // wallUnits = master on/off for the wall row; wallCabs = independent upper-cabinet list along the wall
    A: { wallUnits: true,  units: [ {type, width, handle?:'L'|'R'}, ... ], wallCabs: [ {type, width}, ... ] },  // back wall
    B: { wallUnits: false, units: [ ... ], wallCabs: [ ... ] },                 // left wall  (L, U)
    C: { wallUnits: false, units: [ ... ], wallCabs: [ ... ] },                 // right wall (U)
  },
  window:  { enabled, wall:'back'|'left'|'right', centre, width, height, sill },
  doorway: { enabled, wall, centre, width, height },
  island:  { enabled, length, depth, overhang },
}
```

**Unit types:** `door`, `drawers`, `sink`, `oven` (oven cabinet), `range` (range cooker),
`dishwasher`, `gap` (open space — no cabinet/worktop/wall unit), `corner` (blank/dead corner —
filler panel, no door/handle, worktop over; placed where two runs meet), `fridge` (tall), `larder` (tall),
`ovenTower` (tall). Each unit has a `width` (m); `door`/`fridge`/`larder` units take an optional `handle: 'L'|'R'` (default `R`).

**Auto-derived from the spec when building:**
- Worktop segment over each base unit (not over `range`, tall units, or `gap`).
- Hob + extractor over `oven`; extractor over `range`.
- Sink basin + gooseneck tap over `sink`.
- Wall cupboards over eligible base units when the run's `wallUnits` is on — **skipped** over
  windows, doorways, sinks, tall units and gaps. Under-cabinet LED strip included.
- Window/doorway are cut as openings in their wall; island is a freestanding block with optional
  breakfast-bar overhang.

### 5.2 `state.colors` (the colour scheme)
Six recolourable roles, each a hex string:
`base`, `wall` (wall cupboards), `worktop`, `walls`, `floor`, `handles`.
Applied via the shared material per role: `mats[role].color.set(hex)`.

**Defaults:** Hague Blue base, Cornforth White wall cupboards, Wimborne White worktop,
Pointing walls, Purbeck Stone floor, Railings handles.

### 5.3 Export/import bundle (JSON)
```js
{ spec, colors, tod, app:'KitchenStudio', version:3 }
```
`tod` = time-of-day (0..1). Exported as `kitchen-design.json`; importing restores layout,
colours and lighting.

---

## 6. Features (current)

**Design tab**
- Layout: single / L / U.
- Ceiling height; worktop end overhang (mm); 15mm end panels toggle; global handle width (mm).
- Per-run unit editor: add, remove, reorder (↑/↓), set width, type and handle side (L/R per door); per-run wall-cupboards toggle; live run length.
- Window: enable, wall, offset along wall, width, height, sill.
- Doorway: enable, wall, offset, width, height.
- Island: enable, length, depth, breakfast-bar overhang.
- Save/load: Export JSON, Import JSON, Reset design. Auto-saves to browser.

**Colours tab**
- Six surface rows; click a surface to open an inline dropdown.
- Full **301-colour** F&B palette, grouped (Whites & Neutrals, Greys, Greens, Blues,
  Yellows & Earth, Pinks & Reds, Browns & Drabs, Blacks). Each row shows swatch + name + No.
- **Search** by name or number; **live hover-preview** on the 3D model (reverts if you don't click).
- Save named schemes; quick **A/B compare**.

**Scene tab**
- Time-of-day slider; Day/Evening quick toggle (physical Sky + sun).
- Photoreal toggle (GTAO ambient occlusion + subtle bloom + SMAA), Styling props toggle.
- Recentre view; Snapshot (PNG download); Hide panel.

**3D**
- Orbit/zoom/pan; rounded-edge joinery; procedural PBR materials; flat green lawn seen through windows.

---

## 7. Dimensions reference (metres)

| Constant | Value | Meaning |
|---|---|---|
| `BASE_DEPTH` | 0.60 | base cabinet depth |
| `PLINTH_H` | 0.12 | plinth/kickboard height |
| `COUNTER_H` | 0.92 | worktop top height |
| `WT_TH` | 0.04 | worktop thickness |
| `DOOR_TOP` | 0.88 | top of base doors (COUNTER_H − WT_TH) |
| `WALL_DEPTH` | 0.34 | wall cupboard depth |
| `WALL_Y0 / WALL_Y1` | 1.50 / 2.08 | wall cupboard bottom / top |
| `WALL_T` | 0.12 | room wall thickness |
| `TALL_H` | 2.00 | tall-unit height |
| `TALL_DEPTH` | 0.60 | tall-unit depth |
| `REVEAL` | 0.004 | gap between adjacent fronts |
| ceiling (default) | 2.60 | room height |

---

## 8. Persistence

- **localStorage `kds_spec_v1`** — `{spec, colors, tod}` auto-saved on change; loaded on boot.
- **localStorage `kds_presets_v1`** — saved colour schemes (presets list).
- **File** — `kitchen-design.json` via Export/Import (portable, shareable).

---

## 9. Palette notes

- **301 of 303** colours from `farrow-ball.com/paint/all-paint-colours` (pages 1–4). Two could
  not be captured because the site renders swatches as images and the fetch truncates the listing.
- Names and **No.** are exact from the site. **Hex values are approximations** (no official RGB
  published). They convey mood on screen; always confirm against a physical sample.

---

## 10. Known limitations / caveats

- Base cabinets still render across a **doorway** that sits along a run — drop a **Gap** unit at
  the doorway position (matching width) to clear them.
- No collision/overlap checks (e.g. island vs runs, units exceeding wall length).
- Camera only reframes on layout change or **Recentre** (kept stable during small edits).
- Extractor hood may slightly overlap an adjacent wall cupboard edge.
- Large single file; all logic in one `<script>`.

---

## 11. Roadmap / ideas

- Pre-fill the spec from a described real kitchen (measured template).
- Peninsula option; more appliance detail; worktop finish presets (wood/stone/laminate); brushed-brass handle finish; herringbone floor.
- Optional: screen-space reflections, depth-of-field.
- Graduate to a Vite project if scope grows (see §2).

---

## 12. Changelog (summary)

- **v1** — L-shaped kitchen, 6 recolourable surfaces, ~50 F&B colours, presets, day/evening, snapshot.
- **v2** — Photoreal pass: procedural PBR textures, physical Sky + sun + window light, GTAO/bloom/SMAA, rounded edges, richer detail + styling props.
- **palette** — Expanded to the full **301**-colour F&B range; colour grid replaced by searchable per-surface dropdown with live hover-preview.
- **v3** — Data-driven **spec** model + Design tab (layout single/L/U, per-run unit editor incl. tall units & gaps, window/doorway/island), tabbed UI, JSON export/import, localStorage autosave. (Palette was briefly reduced during this rewrite and then **restored to 301**.)
- **end panels at gaps** — End panels now also cap the exposed cabinet end wherever a run terminates at an internal `gap` unit (e.g. a run stopping short of a corner before a doorway), not only at a run's outer start/end. The worktop extends to overhang these gap-facing panels. Outer-end rule unchanged: panels appear on genuinely exposed ends; inside corners and wall joins get none.
- **Plan tab** — New fourth tab producing scale CAD-style **SVG** drawings generated from the same `spec`: a top-down **floor plan** (poché walls, cabinet footprints, tall-unit hatching, appliance symbols, window double-line, doorway swing arc, island, full dimensioning — per-unit + run totals + overall room) and **front elevations** for each present run (plinth/carcass/worktop/wall units/tall units/appliance faces, openings, per-unit width + height dims). Greyscale style, title block + 1 m scale bar. Drawings render in a white "sheet" overlay over the 3D scene; controls (view toggle, export) live in the side panel. Export via **Print/PDF** (print CSS) and **PNG** (SVG serialised to canvas). Static (no click-to-edit). Sink unit reduced to a single door; under-cabinet LED z-position bug fixed; styling props default off.
- **Plan refinements** — Left-wall elevation now mirrored to read as viewed from inside the room (front→back), the correct mirror of the right-wall elevation. Print forced to **A4 landscape** with the drawing scaled to fill the page. Title block expanded to a two-column block carrying **customer name + address** (new `spec.client` fields, edited in the Plan tab, autosaved and included in JSON export/import).
- **Corner (blank/dead) unit** — New `corner` unit type for the spot where two runs meet: a plain filler (no door/handle) with worktop over it, instead of a door cabinet. Renders as a flat panel in 3D, a blank panel in elevations, and a distinct grey **DEAD CORNER** block on the plan. No wall cupboard is placed above it. Set the corner-owning unit (an end unit of Run A) to this type.
- **Side-run corner ownership** — If the **first unit** of Run B or Run C is a `corner`, that side run now *claims* the corner: it extends to the back wall, Run A shifts clear, and the room widens by `BASE_DEPTH` (600 mm) on that side so the corner fits. Implemented via `cornerGeom()` (claims/offsets/room size), shared by `buildKitchen` (separate full-width back-wall frame; Run A frame gains an `sx` shift) and `planModel`/plan dims. Default behaviour (Run A owns the corners) is unchanged when no side corner is placed.- **Independent wall cupboards** — Wall (upper) cabinets are no longer slaved 1:1 to base units. Each run has its own editable **`wallCabs`** list (own widths/positions) with types **door / extractor / bridging / gap**, edited in a per-run "Wall cupboards" sub-editor under the base units. The base oven now keeps only the hob; extraction is an `extractor` wall unit. Existing designs migrate automatically (`migrateWallCabs()`: base door/drawers/dishwasher → door, oven/range → extractor, others → gap; runs with `wallUnits:false` get an empty list). Wall units render in 3D, in elevations (door/extractor/bridging) and as dashed overlays on the plan. Toggling a run's wall row on auto-seeds the list from the base units.