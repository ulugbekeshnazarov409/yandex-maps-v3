# yandex-maps-v3

An [agent skill](https://www.skills.sh) for the **Yandex Maps JavaScript API v3** (`ymaps3`)
— a complete reference and integration guide for building Yandex maps in web / React /
Next.js apps.

## Install

```bash
npx skills add ulugbekeshnazarov409/yandex-maps-v3
```

## What it covers

- Script loader + API key, and the **`[longitude, latitude]`** coordinate order (v3 quirk)
- `YMap`, layers & data sources, markers (custom DOM + `YMapDefaultMarker`)
- Controls (zoom, geolocation, buttons, `YMapOpenMapsButton`), `YMapListener` events
- Hints & popups, the `@yandex/ymaps3-clusterer` marker clusterer
- Themes (dark/light), geolocation, behaviors
- The **React / Next.js `reactify` pattern** (App Router, SSR-safe dynamic import)

## Structure

```
skills/yandex-maps-v3/
├── SKILL.md                  # entry point (name + description + usage)
└── reference/
    ├── api.md                # full ymaps3 API surface
    ├── packages.md           # add-on packages (clusterer, controls, reactify…)
    └── recipes.md            # copy-paste integration recipes
```

Use it whenever building, debugging, or extending a Yandex map in a web/React/Next.js app
— adding markers, popups, clustering, controls, geocoding, or fixing the lng/lat order,
the API key, `ymaps3.ready`, or `reactify.bindTo`.
