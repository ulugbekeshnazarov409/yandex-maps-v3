# Yandex Maps v3 — importable packages (`ymaps3.import('@yandex/...')`)

The core (`window.ymaps3`) ships `YMap`, layers, `YMapMarker`, `YMapFeature`,
controls (`YMapControls`, `YMapControlButton`, `YMapZoomControl`,
`YMapGeolocationControl`, `YMapScaleControl`), and `YMapListener`. Extra
features come from packages loaded at runtime via `ymaps3.import(...)` (or npm +
reactify). Packages do **not** guarantee backward compatibility — pin versions.

| Package | Exports / purpose |
|---|---|
| **@yandex/ymaps3-reactify** | `reactify.bindTo(React, ReactDOM)` → `reactify.module(ymaps3)` returns React components for every entity. `reactify.useDefault(value, deps?)` sets a prop once (like `defaultValue`). **Required for React/Next.js.** |
| **@yandex/ymaps3-types** | TypeScript types for the whole API (dev dependency). Map `tsconfig` `paths: { "ymaps3": [".../@yandex/ymaps3-types"] }`. |
| **@yandex/ymaps3-clusterer** | `YMapClusterer`, `clusterByGrid({gridSize})`. Marker clustering for many points. `new YMapClusterer({method, features, marker, cluster})`. |
| **@yandex/ymaps3-default-ui-theme** | `YMapDefaultMarker` (styled pin: color, iconName, title, subtitle, size, popup), default UI controls styling, `YMapDefaultMarker` icons. |
| **@yandex/ymaps3-hint** | `YMapHint`, `YMapHintContext` — hover tooltips bound to a feature's `hint` property. |
| **@yandex/ymaps3-cartesian-projection** | Cartesian (flat) projection for non-geographic / custom coordinate systems. |
| **@yandex/ymaps3-web-mercator-projection** | Web-Mercator projection helper. |

## React import pattern

```ts
const r = await loadYmaps() // returns { ymaps3, reactify }
const { YMapClusterer, clusterByGrid } =
  r.reactify.module(await r.ymaps3.import("@yandex/ymaps3-clusterer"))
const { YMapDefaultMarker } =
  r.reactify.module(await r.ymaps3.import("@yandex/ymaps3-default-ui-theme"))
const { YMapHint, YMapHintContext } =
  r.reactify.module(await r.ymaps3.import("@yandex/ymaps3-hint"))
```

## Vanilla import pattern

```js
const { YMapClusterer, clusterByGrid } = await ymaps3.import("@yandex/ymaps3-clusterer")
const { YMapDefaultMarker } = await ymaps3.import("@yandex/ymaps3-default-ui-theme")
```

## Third-party React wrapper (alternative to hand-rolled reactify)

`ymap3-components` (npm) wraps reactify for you: `<YMapComponentsProvider apikey>`
+ `<YMap>`, `<YMapMarker>`, etc. Convenient, but the hand-rolled loader in
`recipes.md` gives full control and avoids an extra dependency.

## Companion (mobile)

For native mobile (Android/iOS/React Native) Yandex Maps, this is a DIFFERENT
SDK — use the **`yandex-mapkit-sdk`** skill, not this one. `yandex-maps-v3` is
web/JS only.
