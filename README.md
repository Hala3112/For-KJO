# KJO Marine Tracker

Live vessel-tracking web client for **Al-Khafji Joint Operations (KJO)**.
Adapted from the KOC Marine Tracker.

The app renders the live AIS fleet (~21k targets) on a Leaflet map as a **single
canvas overlay** — no per-vessel DOM nodes — with server-driven symbology, vessel
search, popups and a detail drawer, history trails, KJO area-of-interest polygons,
measurement/drawing tools, and an optional Windy weather engine.

**Stack** — React 19 · TypeScript 5.9 · Vite 7 · MUI 7 · TanStack Query 5 ·
React Router 7 · Leaflet 1.9 + esri-leaflet 3 · Axios · Windy Map Forecast API

---

## Quick start

```bash
npm install
cp .env.example .env        # then fill in VITE_WINDY_API_KEY / VITE_BASE_PATH
npm run dev                 # Vite dev server
```

| Script | What it does |
| --- | --- |
| `npm run dev` | Dev server with HMR |
| `npm run build` | `tsc -b` (typecheck) then `vite build` → `dist/` |
| `npm run preview` | Serve the production build locally |
| `npm run lint` | ESLint 9 (flat config, typescript-eslint + react-hooks) |

> The app **cannot boot without `public/config.json`** — `loadConfig()` runs as a
> top-level `await` in `src/main.tsx` and throws on a non-200 response.

---

## Features

**Map**
- Canvas fleet layer with viewport padding, zoom-based decimation (below zoom 13,
  one vessel per ~8px cell — KJO/KOC vessels exempt), and name labels from zoom 11
- Symbology resolved from the **ArcGIS layer's own renderer** (`esriPMS`/`esriSMS`),
  pre-rendered to sprites; heading/course rotation and moving/slow/stopped state
- 4 tile basemaps — gray (CARTO Positron, default), imagery (Esri), OSM, dark
- Toggleable layers: vessels, AOI (with colour picker), basemap service, offshore,
  rigs, maritime WMS, graticule, labels, weather
- Auto-refresh of vessel positions every **60 s**

**Vessels**
- Header search by name / MMSI / IMO (400 ms debounce) with a results panel
- Click-to-select with a 30 px hit tolerance, hover at 14 px; popup + detail drawer
- History trails over 1 h / 2 h / 6 h / 12 h / 24 h shortcuts
- Type filters: All, Cargo, Tanker, Passenger, Fishing, Yachts/Sailing/Military/Tugs,
  Other, **KJO Vessels**, Unknown

**Tools**
- Distance / area measurement, drawing (point, line, polygon), go-to coordinates,
  vessel-type legend, north arrow + scale bar, zoom / reset view

**Weather (opt-in)**
- The Windy engine is **off by default**. Enabling it in the Layers panel persists to
  `localStorage` (`kjo-windy-engine`) and reloads the page; only then are Windy's
  scripts downloaded. Adds the Windy basemap, weather overlays, picker and menu.

---

## Configuration

Two layers. **Runtime wins** — anything in `config.json` overwrites the defaults in
`src/config/env.ts` via `Object.assign`.

### 1. `public/config.json` — runtime, editable on the server without a rebuild

| Key | Purpose |
| --- | --- |
| `apiBaseUrl` | REST API base for master data and vessel history |
| `virtualPath` | Path the app is served from; used by the router basename and `assetPath()` |
| `vesselsLayerUrl` | ArcGIS MapServer layer with live vessel points (`…/MapServer/0`) |
| `aoiLayerUrl` | KJO area-of-interest FeatureServer layer (secured — needs a token) |
| `basemapUrl`, `offshoreLayersUrl`, `rigsLayerUrl` | Optional ArcGIS dynamic map layers |
| `maritimeLayerUrl` | Optional WMS maritime-boundaries layer |
| `portalUrl`, `oauthClientId` | ArcGIS Enterprise portal OAuth; **empty disables sign-in** |
| `arcgisToken` | Static token fallback when OAuth is not configured |
| `proxyUrl`, `proxyUrlPrefix` | Esri proxy; comma-separated list of URL prefixes to route through it |
| `mapCenterLat`, `mapCenterLon`, `mapZoom` | Initial view |

Empty-string URLs are treated as "not configured" and the corresponding layer is
never created — an ArcGIS layer built on an empty URL fires a failing request per
tile on every pan.

### 2. `.env` — build-time only

| Variable | Purpose |
| --- | --- |
| `VITE_BASE_PATH` | Vite `base`. **Must match `virtualPath`** in `config.json` |
| `VITE_WINDY_API_KEY` | Windy Map Forecast API key (only used in Windy mode) |

> The old `VITE_API_BASE_URL`-style scheme is gone — everything runtime lives in
> `config.json`.

---

## Authentication

`ensureArcgisLogin()` runs in `main.tsx` **before React renders**:

1. Returning from the portal with `#access_token=…` → validate `state`, store the
   token in `sessionStorage`, scrub the URL.
2. No valid token → redirect to `{portalUrl}/sharing/rest/oauth2/authorize`
   (implicit flow, `response_type=token`, 8 h lifetime) and halt the boot sequence.
3. `portalUrl` or `oauthClientId` empty → skipped entirely.

Failures are logged and the app still boots — the portal being unreachable must not
take the map down (the AOI layer simply stays empty).

**Implicit flow is deliberate:** the portal sends no CORS headers on `oauth2/token`,
so the authorization-code exchange is blocked from the browser origin.

The portal application must list this app's URL (origin + `virtualPath`) among its
registered redirect URIs, or sign-in fails.

---

## Architecture

```
src/
├── main.tsx                 # loadConfig() → ensureArcgisLogin() → render
├── App.tsx                  # AppProviders → AppRouter
├── app/
│   ├── providers/           # QueryClient + MUI theme + CssBaseline
│   └── router/              # BrowserRouter (basename = virtualPath), single route
├── layouts/AppLayout.tsx    # app bar, logo, debounced vessel search
├── config/env.ts            # AppConfig, loadConfig, assetPath, getProxiedUrl
├── services/                # axios client (runtime baseURL), ArcGIS OAuth
├── theme/                   # KJO palette + MUI theme
├── types/                   # ambient decls: windy, esri-leaflet, polylinedecorator
└── features/vessels/        # the whole product — see its own README.md
    ├── components/{list,map,dialogs}/
    ├── hooks/{queries,screens}/
    ├── services/            # vessel + AOI queries, symbology, canvas & graticule layers
    ├── constants/           # AIS mappings, filters, basemaps, tunables
    └── types/
```

**Data flow** — `components` → `hooks/screens` (orchestration state) →
`hooks/queries` (TanStack Query) → `services` (raw HTTP / map infrastructure).
No React inside `services/`. See `src/features/vessels/README.md` for the feature
rules and the deliberate deviations from them.

### Data sources

| Data | Source |
| --- | --- |
| Vessel positions | `vesselsLayerUrl/query` — paged at 2000/page, capped at 25 pages, via proxy |
| Vessel search | Same layer; `Mmsi`/`Imo` for numeric input, `UPPER(Name) LIKE` otherwise |
| Master data | `GET {apiBaseUrl}/masterdata` |
| Vessel history | `GET {apiBaseUrl}/vesselhistory?mmsi&from&to` |
| AOI polygons | `aoiLayerUrl/query?f=geojson` — **direct, no proxy**, OAuth token attached |

---

## Map runtime gotchas

These are the non-obvious constraints; breaking them fails silently.

- **Two Leaflet builds.** With the Windy engine on, Windy owns the map and it runs
  Leaflet **1.4** (loaded from unpkg). Vector layers built by the bundled Leaflet 1.9
  crash inside the 1.4 renderer and silently kill vessel click/hover. Always create
  map-attached vectors (markers, polylines, layer groups, tiles) through
  **`getMapL()`** in `services/mapRuntime.ts`, never the imported `L`. Plain value
  objects (`LatLng`, `Point`) are duck-typed by both builds and are safe either way.
- **Windy boots once per page.** Toggling the engine calls `window.location.reload()`
  by design. Its scripts are loaded on demand by `useWindyMap` — they are *not* in
  `index.html`.
- **Never use the `Latitude`/`Longitude` attributes for placement.** They are in the
  layer's projected CRS. Positions come from the feature geometry, requested with
  `outSR=4326`.
- **`IsKOC` is the shared GIS field name** across the KOC and KJO services and flags
  the JO's own fleet. It surfaces in the UI as "KJO Vessels" — the attribute name
  stays `IsKOC`.
- **`VESSEL_LAYER_FIELDS` is an explicit allowlist.** Requesting `*` pulls every junk
  attribute for the entire fleet on every 60 s refresh. Keep it in sync with what the
  popup, symbology and rotation logic actually read.
- **Tile layers use `crossOrigin`** so the canvas stays untainted for map screenshots.
- The gray basemap is CARTO Positron, not Esri's World_Light_Gray_Base — the Esri
  raster bakes its own graticule into the ocean tiles, which cannot be turned off.

---

## Deployment

```bash
npm run build     # → dist/
```

Serve `dist/` from the virtual path configured in `VITE_BASE_PATH` / `virtualPath`
(currently `/kjo/vessels/client/`), with an SPA fallback rewriting unknown paths to
`index.html`.

`dist/config.json` ships with the build but is meant to be **edited in place on the
server** — changing endpoints, the map view or the portal client id needs no rebuild.
Changing `virtualPath` *does* need a rebuild, because `VITE_BASE_PATH` is baked in.

---

## Handover TODO

- [ ] Point `apiBaseUrl` / `vesselsLayerUrl` (and the optional basemap/rigs/offshore/
      maritime URLs) at the KJO services — currently inherited KOC dev endpoints
- [ ] Register this app's production URL as a redirect URI on the portal application
      and set the production `portalUrl` / `oauthClientId`
- [ ] Replace the favicons in `public/images/logo/` (the KJO header logo is in place
      at `public/images/logo/kjo.png`)
- [ ] Rotate the Windy API key — the fallback hard-coded in `src/config/env.ts` is
      inherited from the KOC project
- [ ] Confirm the default map centre/zoom (currently the Khafji offshore area,
      28.4389 / 48.643 @ z9)
- [ ] Remove the stale `public/images/vessels/19 - Copy.png` and the leftover
      `public/images/KOC_Logo.png` / `logo.png` once branding is signed off
 
