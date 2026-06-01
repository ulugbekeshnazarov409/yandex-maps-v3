# Search, Suggest, Geocoding & Routing (HTTP APIs alongside ymaps3)

The JS API v3 draws the map; **address search, autocomplete, reverse geocoding, POI
search, and routing are SEPARATE Yandex HTTP API products, each with its OWN API key**
(enable them in the Developer Dashboard). Call them from your **server** (Next.js route
handler) to keep keys clean, cache, and rate-limit — then feed results to the map.

> Coordinate-order trap: the **Geocoder** returns/【takes】 `"longitude latitude"` (lng
> lat, space-separated) — matching ymaps3's `[lng, lat]`. The **Geosuggest** + **Router**
> APIs use `latitude,longitude`. Normalize at the boundary.

## 1. Geosuggest — address autocomplete (type-ahead)

`https://suggest-maps.yandex.ru/v1/suggest?apikey=SUGGEST_KEY&text=<q>&lang=ru_RU`
Useful params: `results` (max), `ll=lng,lat` + `spn` or `bbox` (bias to viewport),
`types=geo,biz`, `print_address=1`, `attrs=uri`.

```ts
// app/api/suggest/route.ts (server)
export async function GET(req: Request) {
  const q = new URL(req.url).searchParams.get("text") ?? "";
  const u = new URL("https://suggest-maps.yandex.ru/v1/suggest");
  u.searchParams.set("apikey", process.env.YANDEX_SUGGEST_KEY!);
  u.searchParams.set("text", q);
  u.searchParams.set("lang", "ru_RU");
  u.searchParams.set("results", "7");
  u.searchParams.set("print_address", "1");
  const r = await fetch(u, { next: { revalidate: 60 } });
  return Response.json(await r.json()); // { results: [{ title:{text}, subtitle?, address?, uri }] }
}
```
Client: **debounce ~250ms**, cache by query, never call per keystroke. To turn a chosen
suggestion into coordinates, pass its address to the Geocoder (below) or use its `uri`
with `geocode=<uri>`.

## 2. Geocoder — address → coordinates (forward)

`https://geocode-maps.yandex.ru/v1/?apikey=GEOCODER_KEY&format=json&geocode=<address>&lang=ru_RU`

```ts
const u = new URL("https://geocode-maps.yandex.ru/v1/");
u.searchParams.set("apikey", process.env.YANDEX_GEOCODER_KEY!);
u.searchParams.set("format", "json");
u.searchParams.set("geocode", "Toshkent, Amir Temur 1");
u.searchParams.set("lang", "ru_RU");
const data = await (await fetch(u)).json();
const pos = data.response.GeoObjectCollection.featureMember[0]?.GeoObject.Point.pos; // "69.28 41.31" = "lng lat"
const [lng, lat] = pos.split(" ").map(Number);                                       // ready for ymaps3
```
`results`/`skip` paginate; `bbox=lng1,lat1~lng2,lat2` + `rspn=1` restricts to an area.

## 3. Reverse geocoding — coordinates → address

Same endpoint, pass `geocode=<lng>,<lat>` (lng first) + `kind`:
```ts
u.searchParams.set("geocode", `${lng},${lat}`);  // lng,lat
u.searchParams.set("kind", "house");             // house | street | district | locality | province | country
// read GeoObject.metaDataProperty.GeoocoderMetaData.text (full address) +
// .Address.Components[] (kind → name): country / province / locality / district / street / house
```
Cache reverse results aggressively (a tile's address rarely changes).

## 4. POI / business search (Places API)

`https://search-maps.yandex.ru/v1/?apikey=PLACES_KEY&text=<q>&lang=ru_RU&type=biz&ll=lng,lat&spn=0.1,0.1&results=20`
Returns GeoJSON `features[]` with `geometry.coordinates` ([lng,lat]) +
`properties.CompanyMetaData` (name, address, categories, hours, phones). Draw each as a
marker (own key/product; bias to the viewport with `ll`+`spn` or `bbox`).

## 5. Routing — build & draw a route

v3 has **no built-in route editor** (v2's `multiRoute` is gone). Get route geometry from
the **Router HTTP API** (separate key/product — driving / pedestrian / masstransit /
truck), then draw it as a `YMapFeature` LineString in the features layer. Router waypoints
are `lat,lon`; the returned geometry you draw must be `[lng, lat]`.

```ts
// server: call the Router API (see Router API docs for the exact endpoint/params for
// your product), get an ordered list of [lng,lat] points + summary (distance, duration).
// Then on the client, draw it:
const line = {
  type: "Feature",
  geometry: { type: "LineString", coordinates: routePoints /* [[lng,lat], ...] */ },
} as const;

// vanilla:
map.addChild(new ymaps3.YMapFeature({
  geometry: line.geometry,
  style: { stroke: [{ color: "#2563eb", width: 5 }] },
}));
// reactify:  <YMapFeature geometry={line.geometry} style={{ stroke: [{ color:"#2563eb", width:5 }] }} />
```
Keep the route in its **own** feature (or a `YMapCollection`); remove it before drawing a
new one. Show `distance` / `duration` from the router summary in your UI.

## 6. Keys & limits

- Each product (JS API, Geocoder, Geosuggest, Places, Router) is enabled separately and
  may bill separately. The JS API URL can carry helper keys, e.g.
  `…/v3/?apikey=<JS>&suggest_apikey=<SUGGEST>`.
- Server-proxy the HTTP APIs: cache (L2/L4), debounce, rate-limit, and never expose the
  Geocoder/Router/Places keys to the browser (only the JS-API key is public).
- Latency budgets (good UX): autocomplete < 150ms, reverse geocode < 300ms, route < 1s.

## Sources
- Geosuggest API: https://yandex.com/maps-api/docs/suggest-api/request.html
- Geocoder API: https://yandex.com/maps-api/docs/geocoder-api/quickstart.html
- Places/Search API: https://yandex.com/maps-api/products/geocoder-api
- v2→v3 migration: https://yandex.com/maps-api/docs/js-api/upgrade/index.html
