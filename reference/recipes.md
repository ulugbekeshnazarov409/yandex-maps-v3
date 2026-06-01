# Yandex Maps v3 — Next.js App Router recipes (copy-paste)

All map code is **client-only**. Load the component with
`next/dynamic({ ssr: false })` and put `'use client'` at the top of the map file.

## 0. Env + dynamic mount

```ts
// any client page/section
import dynamic from "next/dynamic"
const ListingsMap = dynamic(() => import("@/components/map").then(m => m.ListingsMap), {
  ssr: false,
  loading: () => <div className="grid h-full place-items-center">Yuklanmoqda…</div>,
})
```
`.env.local`: `NEXT_PUBLIC_YANDEX_MAPS_JS_API_KEY=...`

## 1. Singleton loader (script inject + reactify, bound to YOUR React)

```ts
"use client"
import * as React from "react"
import ReactDOM from "react-dom"

const API_KEY = process.env.NEXT_PUBLIC_YANDEX_MAPS_JS_API_KEY
declare global { interface Window { ymaps3?: any } }

let promise: Promise<any> | null = null
export function loadYmaps(): Promise<any> {
  if (typeof window === "undefined") return Promise.reject()
  if (promise) return promise
  promise = new Promise((resolve, reject) => {
    const done = async () => {
      try {
        await window.ymaps3.ready
        const reactifyMod = await window.ymaps3.import("@yandex/ymaps3-reactify")
        const reactify = reactifyMod.reactify.bindTo(React, ReactDOM)
        resolve({ ymaps3: window.ymaps3, reactify, comps: reactify.module(window.ymaps3) })
      } catch (e) { reject(e) }
    }
    if (window.ymaps3) return done()
    const id = "ymaps3-script"
    const existing = document.getElementById(id)
    if (existing) { existing.addEventListener("load", done); return }
    const s = document.createElement("script")
    s.id = id
    s.src = `https://api-maps.yandex.ru/v3/?apikey=${API_KEY}&lang=ru_RU`
    s.async = true
    s.onload = done
    s.onerror = reject
    document.head.appendChild(s)
  })
  return promise
}
```
Why a singleton + `bindTo(React, ReactDOM)`: reactify must wrap the *same* React
instance the app renders with, and the script must load exactly once across
mounts/navigations.

## 2. Map component — markers + click popup + theme + fit-bounds

```tsx
"use client"
import * as React from "react"
import { useTheme } from "next-themes"
import { loadYmaps } from "./loader"

type Pin = { id: string; title: string; price: string; lng: number; lat: number }

export function ListingsMap({ pins }: { pins: Pin[] }) {
  const { resolvedTheme } = useTheme()
  const [comps, setComps] = React.useState<any>(null)
  const [failed, setFailed] = React.useState(false)
  const [openId, setOpenId] = React.useState<string | null>(null)

  React.useEffect(() => {
    let alive = true
    loadYmaps().then(r => alive && setComps(r.comps)).catch(() => alive && setFailed(true))
    return () => { alive = false }
  }, [])

  if (failed) return <div className="grid h-full place-items-center">Xarita yuklanmadi</div>
  if (!comps) return <div className="grid h-full place-items-center">Yuklanmoqda…</div>

  const { YMap, YMapDefaultSchemeLayer, YMapDefaultFeaturesLayer, YMapMarker } = comps
  const center = pins.length ? [pins[0].lng, pins[0].lat] : [69.2797, 41.3111] // [lng,lat]

  return (
    <YMap location={{ center, zoom: 11 }} theme={resolvedTheme === "dark" ? "dark" : "light"}>
      <YMapDefaultSchemeLayer />
      <YMapDefaultFeaturesLayer />
      {pins.map(p => (
        <YMapMarker key={p.id} coordinates={[p.lng, p.lat]}
          onClick={() => setOpenId(id => id === p.id ? null : p.id)}>
          <div className="relative -translate-x-1/2 -translate-y-1/2">
            <span className="map-pin"><span className="map-pin-dot" /></span>
            {openId === p.id && (
              <div onClick={e => e.stopPropagation()}
                   className="absolute bottom-8 left-1/2 w-52 -translate-x-1/2 rounded-2xl border bg-popover p-3 shadow-xl">
                <p className="text-sm font-semibold">{p.title}</p>
                <p className="font-bold text-emerald-600">{p.price}</p>
              </div>
            )}
          </div>
        </YMapMarker>
      ))}
    </YMap>
  )
}
```
Popup pattern: the popup lives INSIDE the marker element, so it auto-tracks the
coordinate — no projection math. `stopPropagation` keeps a click inside the card
from toggling it closed.

Pin CSS (Tailwind/global):
```css
.map-pin { display:grid; place-items:center; width:24px; height:24px; }
.map-pin-dot { width:14px; height:14px; border-radius:9999px; background:#10b981;
  box-shadow:0 0 0 4px color-mix(in oklch,#10b981 30%,transparent); }
.map-pin::before { content:""; position:absolute; width:24px; height:24px; border-radius:9999px;
  background:#10b981; opacity:.35; animation:pin-pulse 2s ease-out infinite; }
@keyframes pin-pulse { 0%{transform:scale(.5);opacity:.5} 100%{transform:scale(2.2);opacity:0} }
@media (prefers-reduced-motion: reduce) { .map-pin::before { animation:none } }
```

## 3. Fit map to all markers (imperative, via ref)

```tsx
const mapRef = React.useRef<any>(null)
// <YMap ref={mapRef} ...>
React.useEffect(() => {
  if (!mapRef.current || pins.length === 0) return
  if (pins.length === 1) { mapRef.current.update({ location: { center: [pins[0].lng, pins[0].lat], zoom: 14 } }); return }
  const lngs = pins.map(p => p.lng), lats = pins.map(p => p.lat)
  const bounds = [[Math.min(...lngs), Math.min(...lats)], [Math.max(...lngs), Math.max(...lats)]]
  mapRef.current.update({ location: { bounds, duration: 400 } })
}, [pins])
```

## 4. Controls (zoom, geolocation, open-in-yandex)

```tsx
const { YMapControls, YMapZoomControl, YMapGeolocationControl } = comps
// inside <YMap>:
<YMapControls position="right">
  <YMapZoomControl />
  <YMapGeolocationControl />
</YMapControls>
```
`YMapGeolocationControl` triggers the browser geolocation prompt and recentres.

## 5. Clusterer (many markers)

```tsx
const { YMapClusterer, clusterByGrid } = comps.reactifyImport
  ? comps.reactifyImport
  : null // see note
// Proper React way: build clusterer comps from the reactified import:
const r = await loadYmaps()
const { YMapClusterer, clusterByGrid } =
  r.reactify.module(await r.ymaps3.import("@yandex/ymaps3-clusterer"))

const method = React.useMemo(() => clusterByGrid({ gridSize: 64 }), [])
const features = pins.map(p => ({
  type: "Feature", id: p.id,
  geometry: { type: "Point", coordinates: [p.lng, p.lat] }, properties: {},
}))
const marker = (f: any) => <YMapMarker coordinates={f.geometry.coordinates}><div className="map-pin"><span className="map-pin-dot"/></div></YMapMarker>
const cluster = (coordinates: any, fs: any[]) => (
  <YMapMarker coordinates={coordinates}>
    <div className="cluster-bubble">{fs.length}</div>
  </YMapMarker>
)
// inside <YMap>:
<YMapClusterer method={method} features={features} marker={marker} cluster={cluster} />
```

## 6. Cleanup

reactify unmounts the map automatically when the `<YMap>` component unmounts.
For the **vanilla** API always call `map.destroy()` in the effect cleanup:
```ts
React.useEffect(() => { const map = new ymaps3.YMap(el, opts); return () => map.destroy() }, [])
```

## 7. Geocoding (address → coordinates)

The JS API focuses on the map; for address↔coordinate use the **Geocoder HTTP
API** (same key) server-side:
`https://geocode-maps.yandex.ru/1.x/?apikey=KEY&format=json&geocode=<address>`
→ parse `response.GeoObjectCollection.featureMember[].GeoObject.Point.pos`
(returns `"lng lat"`). Call it from a server route to keep the key usage clean.
