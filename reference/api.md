# Yandex Maps v3 — Full class & type reference

All entities extend **YMapEntity** and are mounted with `map.addChild(entity)` /
removed with `map.removeChild(entity)`. In React (reactify) each is a component.

## Core

| Class | Purpose / key props |
|---|---|
| **YMap** | Map container. `new YMap(el, props)`. props: `location {center:[lng,lat], zoom \| bounds:[[lng,lat],[lng,lat]]}`, `mode:'vector'\|'raster'`, `theme:'light'\|'dark'`, `behaviors:string[]`, `margin`, `camera {tilt, azimuth}`, `zoomRange {min,max}`, `restrictMapArea`, `projection`, `showScaleInCopyrights`. Methods: `addChild`, `removeChild`, `update(props)`, `setLocation`, `setMode`, `setBehaviors`, `setCamera`, `setMargin`, `destroy()`. |
| **YMapEntity** | Base class for everything added to the map. |
| **YMapCollection** | Group multiple entities; add/remove as one. |
| **YMapGroupEntity / GenericGroupEntity** | Base for entities with children. |
| **YMapComplexEntity / GenericComplexEntity** | Base for entities composed of sub-entities. |
| **YMapContainer** | DOM container wrapper entity. |
| **YMapContext / YMapContextProvider** | Share state down the entity tree. |

## Layers & data sources

| Class | Purpose |
|---|---|
| **YMapDefaultSchemeLayer** | The Yandex base map (vector tiles). Add first. Props: `theme`, `customization`. |
| **YMapDefaultFeaturesLayer** | Auto-adds a `markers`-type `YMapLayer` + the default feature datasource. Needed before adding `YMapMarker`/`YMapFeature`. |
| **YMapLayer** | A render layer. `{type:'markers' \| 'features' \| ...}`. |
| **YMapTileDataSource** | Raster/tile data provider. |
| **YMapFeatureDataSource** | Vector feature data provider (`{id:'...'}`); markers reference it via `source`. |

## Visual objects

| Class | Key props |
|---|---|
| **YMapMarker** | `new YMapMarker(props, htmlElement)`. props: `coordinates:[lng,lat]`, `draggable`, `mapFollowsOnDrag`, `source`, `zIndex`, `properties`, `onClick`, `onDragStart/Move/End`, `onDoubleClick`. The DOM element is anchored at the coordinate (center with CSS transforms). |
| **YMapDefaultMarker** | (`@yandex/ymaps3-default-ui-theme`) styled pin: `coordinates`, `color`, `iconName`, `title`, `subtitle`, `size:'micro'\|'small'\|'normal'`, `popup`, `onClick`, `draggable`. |
| **YMapFeature** | Polygon/line/point geometry: `geometry` (GeoJSON), `style {stroke:[{color,width}], fill, fillOpacity}`, `properties`, `onClick`. |
| **YMapHotspot** | Interactive area used for feature hover/selection. |

## Controls

| Class | Notes |
|---|---|
| **YMapControls** | Container. `{position:'top'\|'bottom'\|'left'\|'right'\|'top left'\|'top right'\|'bottom left'\|'bottom right', orientation?}`. Add control children. |
| **YMapControlButton** | `{text, onClick, element?, disabled?}`. |
| **YMapControlCommonButton** | Standard styled button. |
| **YMapZoomControl** | `+/−` zoom buttons. |
| **YMapGeolocationControl** | "find me" button (uses browser geolocation, centers map). |
| **YMapScaleControl** | Scale ruler. |
| **YMapOpenMapsButton** | Deep-links the current view into Yandex Maps (web/app). Place inside `YMapControls`. |
| **YMapControl** | Base control class. |

## Events

| Class | Notes |
|---|---|
| **YMapListener** | `{layer:'any', onClick, onUpdate, onMouseMove, ...}`. DOM handler signature `(object, event)` where `object = {type, entity} \| undefined` and `event = {coordinates:[lng,lat], screenCoordinates:[x,y]}`. Map handlers: `onUpdate({location,camera,mapInAction})`, `onResize({size})`, `onActionStart/End`. `listener.update({onClick:null})` removes a handler. |

Handler list: `onClick onDblClick onFastClick onRightDblClick onContextMenu
onMouseDown onMouseUp onMouseEnter onMouseLeave onMouseMove onPointerDown
onPointerMove onPointerUp onPointerCancel onTouchStart onTouchMove onTouchEnd
onTouchCancel onUpdate onResize onActionStart onActionEnd onIndoorPlansChanged`.

## Types / interfaces

`LngLat = [number, number]` (**[lng, lat]**) · `LngLatBounds = [LngLat, LngLat]`
(SW, NE) · `Location {center?, zoom?, bounds?, tilt?, azimuth?, duration?}` ·
`Camera {tilt, azimuth}` · `YMapTheme = 'light' | 'dark'` ·
`MapMode = 'vector' | 'raster'` · `ZoomRange {min,max}` · `TiltRange` ·
`Feature {type:'Feature', id, geometry:{type:'Point', coordinates:LngLat}, properties}` ·
`Geometry` (GeoJSON) · `DrawingStyle`, `StrokeStyle` · `Projection` ·
`MapEvent`, `DomEvent`, `MouseDomEvent`, `PointerDomEvent`, `TouchDomEvent` ·
`Config` (whole-API config) · `IndoorPlan`, `IndoorLevel`.

## Behaviors (interaction)

`drag · pinchZoom · scrollZoom · dblClick · magnifier · oneFingerZoom ·
mouseRotate · mouseTilt · pinchRotate · panTilt`. Set via `behaviors` prop or
`map.setBehaviors([...])`. Default ≈ `['drag','pinchZoom','mouseTilt']`.
