---
name: yandex-maps-v3
description: >-
  Complete reference and integration guide for the Yandex Maps JavaScript API
  v3 (ymaps3) — script loader + API key, the [longitude, latitude] coordinate
  order, YMap / layers / data sources, markers (custom DOM + YMapDefaultMarker),
  controls (zoom, geolocation, buttons, YMapOpenMapsButton), YMapListener events,
  hints & popups, the @yandex/ymaps3-clusterer marker clusterer, themes
  (dark/light), geolocation, behaviors, and the React/Next.js reactify pattern
  (App Router, SSR-safe dynamic import). Use whenever building, debugging, or
  extending a Yandex map in a web/React/Next.js app — adding markers, popups,
  clustering, controls, geocoding, or fixing the lng/lat order, the api key,
  ymaps3.ready, or reactify.bindTo.
---

# Yandex Maps JavaScript API v3 (ymaps3)

The modern (v3) Yandex Maps web SDK. Vector-first, promise-based, modular
(extra features loaded via `ymaps3.import('@yandex/...')`), with a first-class
React wrapper (reactify).

> **THE #1 GOTCHA:** Yandex v3 uses **`[longitude, latitude]`** order
> everywhere (`center`, `coordinates`, `bounds`). This is the OPPOSITE of
> Leaflet/Google (`[lat, lng]`). Mixing them puts Tashkent in the ocean.
> Toshkent center = `[69.2797, 41.3111]`.

For exhaustive detail load the bundled references:
- `reference/api.md` — every class/type with props.
- `reference/recipes.md` — copy-paste Next.js App Router patterns (loader,
  markers+popup, clusterer, controls, geolocation, theme sync, fit-bounds).
- `reference/packages.md` — importable `@yandex/*` modules.

---

## 1. API key

Get a **"JavaScript API and Geocoder"** key at
https://developer.tech.yandex.com/ (or developer.tech.yandex.ru).

- Activation takes **up to 15 minutes**.
- Fill the **"Restriction by HTTP Referer"** field with your domains
  (`localhost`, `*.yourdomain.com`). Without it the key is rejected.
- The key is public (goes in client URL) — restrict by referer, not by secrecy.
- Store as `NEXT_PUBLIC_YANDEX_MAPS_JS_API_KEY` (Next.js) — it must be exposed
  to the browser.

**Local development:** add `localhost` to the key's **"Restriction by HTTP
Referer"** field in the developer cabinet (changes take ~15 min). Serve over a
real HTTP origin, not `file://` — `npx serve` (→ `http://localhost:3000`) or
`python -m http.server` (→ `:8000`). In Next.js `npm run dev` already serves on
`localhost`, so just whitelist `localhost` (and your prod domain).

## 2. Loading the script

```html
<script src="https://api-maps.yandex.ru/v3/?apikey=YOUR_KEY&lang=en_US"></script>
```

`lang` values: `en_US`, `ru_RU`, `uz_UZ`, `tr_TR`, `uk_UA`, etc. For
Uzbekistan, `ru_RU` usually has the most complete map labels; `uz_UZ` is the
localized option.

After the script tag, `window.ymaps3` exists but is NOT ready until its
promise resolves:

```js
await ymaps3.ready; // waits until the main module is fully loaded
```

## 3. Vanilla quickstart

```js
await ymaps3.ready;
const { YMap, YMapDefaultSchemeLayer, YMapDefaultFeaturesLayer, YMapMarker } = ymaps3;

const map = new YMap(
  document.getElementById('map'),       // a visible, NON-zero-sized element
  {
    location: { center: [69.2797, 41.3111], zoom: 11 }, // [lng, lat]!
    theme: 'light',                     // 'light' | 'dark'
    mode: 'vector',                     // 'vector' | 'raster'
  }
);

map.addChild(new YMapDefaultSchemeLayer());   // the Yandex base map tiles
map.addChild(new YMapDefaultFeaturesLayer());  // hosts markers/features

// A marker with custom HTML content (anchored at its coordinates):
const el = document.createElement('div');
el.className = 'pin';
map.addChild(new YMapMarker({ coordinates: [69.2797, 41.3111] }, el));
```

Key rules:
- The container must be **visible and non-zero-sized** before `new YMap` (give
  it explicit width/height). Otherwise the map renders blank.
- The map fills its bounding box and tracks size via `ResizeObserver`.
- Always call **`map.destroy()`** on teardown (React unmount) — the DOM element
  stays, the map instance is freed.

## 4. Map: location, theme, mode, behaviors

`location` accepts EITHER center+zoom OR bounds:
```js
{ location: { center: [lng, lat], zoom: 11 } }
{ location: { bounds: [[lng1, lat1], [lng2, lat2]] } }  // SW, NE corners
```

Imperative updates (do NOT recreate the map):
```js
map.update({ location: { center: [lng, lat], zoom: 13, duration: 400 } });
map.setLocation({ bounds, duration: 300 });   // animate to fit bounds
map.setMode('raster');                          // 'vector' | 'raster'
map.setBehaviors(['drag', 'pinchZoom', 'scrollZoom', 'dblClick']);
```

- **theme**: `'light'` | `'dark'`. Sync with your app theme.
- **behaviors** (interaction): `drag`, `pinchZoom`, `scrollZoom`, `dblClick`,
  `magnifier`, `oneFingerZoom`, `mouseRotate`, `mouseTilt`, `pinchRotate`,
  `panTilt`. Default ≈ `['drag','pinchZoom','mouseTilt']`.
- **camera**: `zoom`, `tilt`, `azimuth` (rotation). `zoomRange: {min,max}`.

## 5. The entity model

Everything on the map is a **YMapEntity** added via `map.addChild(entity)` and
removed via `map.removeChild(entity)`. Composition tree:

```
YMap
├── YMapDefaultSchemeLayer        // base tiles
├── YMapDefaultFeaturesLayer      // default datasource + markers/features layer
├── YMapMarker / YMapDefaultMarker
├── YMapFeature                   // polygons/lines/points (GeoJSON geometry)
├── YMapControls (position)       // └─ YMapControlButton, YMapZoomControl, …
├── YMapListener                  // events
└── YMapClusterer                 // (from @yandex/ymaps3-clusterer)
```

Manual layering (advanced): `YMapTileDataSource` + `YMapLayer({type:'markers'})`
+ `YMapFeatureDataSource`. For 95% of cases the two Default* layers are enough.

## 6. Markers

**Custom DOM marker** (full control over the pin element):
```js
new YMapMarker(
  { coordinates: [lng, lat], draggable: false, zIndex: 10,
    onClick: () => {}, onDragMove: (coords) => {} },
  htmlElement                 // your pin DOM; centered on the coordinate
);
```
The element is positioned with its origin at the coordinate — offset it with
CSS `transform: translate(-50%, -50%)` to center, or `-100%` vertically to make
a "tip-at-point" pin.

**YMapDefaultMarker** (styled, from `@yandex/ymaps3-default-ui-theme`):
```js
const { YMapDefaultMarker } = await ymaps3.import('@yandex/ymaps3-default-ui-theme');
new YMapDefaultMarker({
  coordinates: [lng, lat],
  color: '#10b981', iconName: 'landmark',
  title: 'Title', subtitle: 'Subtitle',
  size: 'normal',               // 'small' | 'normal' | 'micro'
  popup: { content: () => el }, // built-in popup
  onClick: () => {},
});
```

## 7. Controls

```js
const { YMapControls, YMapControlButton, YMapZoomControl, YMapGeolocationControl } = ymaps3;

const controls = new YMapControls({ position: 'right' });
controls.addChild(new YMapZoomControl({}));
controls.addChild(new YMapGeolocationControl({}));
controls.addChild(new YMapControlButton({ text: 'Reset', onClick: () => map.update({location}) }));
map.addChild(controls);
```
- `position`: `'top' | 'bottom' | 'left' | 'right' | 'top left' | 'top right' |
  'bottom left' | 'bottom right'`.
- **YMapOpenMapsButton** — a button that opens the current view in Yandex Maps
  (deep-link); add inside `YMapControls`. Good for "Open in Yandex Maps".

## 8. Events — YMapListener

```js
const listener = new YMapListener({
  layer: 'any',
  onClick: (object, event) => {
    // object: { type, entity } | undefined (undefined = map background)
    // event:  { coordinates: [lng,lat], screenCoordinates: [x,y] }
  },
  onUpdate: ({ location, camera, mapInAction }) => {},  // pan/zoom
  onMouseMove: (o, e) => {},
});
map.addChild(listener);
listener.update({ onClick: null });   // remove one handler
```
Handlers: `onClick`, `onDblClick`, `onFastClick`, `onContextMenu`,
`onMouseDown/Up/Enter/Leave/Move`, `onPointerDown/Move/Up/Cancel`,
`onTouchStart/Move/End/Cancel`, `onUpdate`, `onResize`, `onActionStart/End`.
Removed automatically when the listener entity is removed.

## 9. Hints & popups

- **Hints** (hover tooltips): `@yandex/ymaps3-hint` → `YMapHint` + `YMapHintContext`.
  Features expose a `hint` property; `YMapHint` renders it.
- **Popups** (click): use `YMapDefaultMarker`'s `popup`, or build your own by
  rendering an absolutely-positioned card INSIDE a custom `YMapMarker` element
  toggled on click (it auto-tracks the coordinate — no manual projection). This
  is the simplest popup pattern in React (see `reference/recipes.md`).

## 10. Clustering — @yandex/ymaps3-clusterer

```js
const { YMapClusterer, clusterByGrid } = await ymaps3.import('@yandex/ymaps3-clusterer');

const features = [
  { type: 'Feature', id: '1', geometry: { type: 'Point', coordinates: [lng, lat] }, properties: {} },
  // …
];
const clusterer = new YMapClusterer({
  method: clusterByGrid({ gridSize: 64 }),
  features,
  marker: (feature) => new YMapMarker({ coordinates: feature.geometry.coordinates }, pinEl()),
  cluster: (coordinates, features) =>
    new YMapMarker({ coordinates, onClick: () => map.update({ location: { bounds: getBounds(features) } }) },
      circle(features.length)),
});
map.addChild(clusterer);
```
Use clustering when you may render many markers; `gridSize` (px) controls
cluster radius. React version: `reactify.module(await ymaps3.import(...))`.

## 11. React / Next.js (reactify) — the production pattern

ymaps3 is imperative; the official **reactify** wrapper turns every entity into
a React component. **It is client-only** — never import at module top level in a
Server Component; load via `next/dynamic({ ssr: false })`.

```ts
// loader: bind reactify to YOUR React instance, then build components
const [react] = await Promise.all([ymaps3.import('@yandex/ymaps3-reactify'), ymaps3.ready]);
const reactify = react.reactify.bindTo(React, ReactDOM);
const { YMap, YMapDefaultSchemeLayer, YMapDefaultFeaturesLayer, YMapMarker, YMapControls, YMapListener } =
  reactify.module(ymaps3);
```

```tsx
<YMap location={reactify.useDefault({ center: [lng, lat], zoom: 11 })} theme="dark">
  <YMapDefaultSchemeLayer />
  <YMapDefaultFeaturesLayer />
  <YMapMarker coordinates={[lng, lat]} onClick={...}>
    <div className="pin" />
  </YMapMarker>
</YMap>
```

**`reactify.useDefault(value)`** sets a prop ONCE (like `defaultValue`) so React
re-renders don't fight the imperative map state. Use it for `location` and other
"initial" props you don't want to force every render. Pass `deps` as 2nd arg to
re-apply.

Full SSR-safe Next.js component (script injection + cleanup + theme + markers +
popup + fit-bounds) is in `reference/recipes.md`.

## 12. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Markers in the ocean / wrong place | Coordinates passed as `[lat, lng]`. Yandex needs **`[lng, lat]`**. |
| Blank/grey map | Container has zero size before `new YMap`; give it explicit width/height. Or invalid/inactive API key (wait 15 min, set HTTP-Referer). |
| `ymaps3 is not defined` | Script not loaded yet — `await ymaps3.ready`; load script client-side only. |
| Works once, breaks on navigation | Map not destroyed on unmount — call `map.destroy()` (or rely on reactify unmount). |
| Hydration / "window is not defined" | Importing reactify/ymaps3 in SSR — wrap in `dynamic(..., { ssr: false })`, `'use client'`. |
| Reactify components render nothing | `reactify.bindTo` must use the SAME React/ReactDOM the app uses; build the module once and memoize. |
| Many markers janky | Use `@yandex/ymaps3-clusterer`. |

## Sources
- Quick start: https://yandex.com/maps-api/docs/js-api/quickstart.html
- Map: https://yandex.com/maps-api/docs/js-api/dg/concepts/map.html
- React (reactify): https://yandex.com/maps-api/docs/js-api/dg/concepts/integrations/reactify.html
- Reference: https://yandex.com/maps-api/docs/js-api/ref/index.html
- Packages: https://yandex.com/maps-api/docs/js-api/ref/packages/index.html
- Clusterer example: https://yandex.com/dev/jsapi30/doc/en/examples/cases/markers-clusterizer
- Controls button: https://yandex.ru/maps-api/docs/js-api/object/controls/buttons/YMapOpenMapsButton.html
