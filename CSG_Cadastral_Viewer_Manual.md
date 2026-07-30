# CSG Cadastral Viewer — Implementation Manual

A record of everything built, fixed, and learned while developing the single-file
Leaflet/esri-leaflet cadastral GIS viewer against South Africa's Chief
Surveyor-General ArcGIS REST service. Written so the same features and fixes can
be reimplemented in a different codebase or environment without repeating the
same investigation from scratch.

This is not a changelog of what changed — it's a manual of **how** and **why**,
with working code, so the reasoning transfers even if the target app's structure
doesn't match this one exactly.

---

## 1. What this app is

A single HTML file (`index.html`) containing an entire Leaflet-based cadastral
map viewer:

- **Map engine**: Leaflet 1.9.4 + esri-leaflet 3.0.12, all loaded via CDN.
- **Data source**: `https://csggis.dlrrd.gov.za/server/rest/services/CSGSearch/MapServer`
  — a public ArcGIS REST FeatureServer/MapServer with one layer per cadastral
  data type (Erven, Holding, Farm Portion, Servitudes, Province, etc).
- **Export libraries**: JSZip 3.10.1 (KMZ/Shapefile zipping), Turf.js 6.5.0
  (geometry ops), @tmcw/togeojson 5.8.1 (KML/KMZ import), sql.js (loaded
  dynamically, only when a GeoPackage export is requested — this keeps the
  base page light).
- **No build step, no server** — everything is inline `<script>`/`<style>` in
  one HTML file. This constrains some choices below (e.g. why exports are
  built by hand with `DataView`/string templates instead of a library).

If reimplementing in a different stack (React app, server-rendered app, etc),
the concepts below still apply — only the plumbing (how you fetch layer
defs, how you bind click handlers) will differ.

---

## 2. Layer catalogue pattern

Every layer is one entry in a flat array, not scattered across separate code
paths per layer. This is the single biggest thing that kept the codebase
manageable as more layers/behaviors were added:

```js
const LAYER_DEFS = [
  { key:'erven', id:2, name:'Erven', cat:'landparcel', type:'polygon',
    color:'#00ff00', weight:1.0, fillOpacity:0, minZoom:14,
    defaultChecked:false, secondary:true, labelField:'TAG_VALUE',
    pane:'paneParcels' },
  { key:'holding', id:4, name:'Holding', cat:'landparcel', type:'polygon',
    color:'#ff9900', weight:1.0, fillOpacity:0, minZoom:13,
    defaultChecked:false, secondary:true, labelField:'TAG_VALUE',
    pane:'paneParcels' },
  // ...one entry per ArcGIS sub-layer id...
  { key:'province', id:14, name:'Province', cat:'admin', type:'polygon',
    color:'#ff0000', weight:4.0, fillOpacity:0, minZoom:0,
    alwaysVisible:true, defaultChecked:true, labelField:'PROVINCE',
    pane:'paneProvinces' },
];
```

**Lesson**: every feature added later (secondary/scale-gating, labeling,
export symbology, the attribute CSV) was implemented as *one more property on
this object* plus *one generic function reading it*, never as a per-layer
`if (key === 'erven')` branch. That pattern is worth preserving in a port —
it's what let 12 layers × 4 export formats × labeling × CSV stay manageable in
one file.

**Custom Leaflet panes for z-order** — declare panes explicitly rather than
relying on layer-add order, so styling/z-index stays correct regardless of
what order layers happen to load:

```js
map.createPane('paneProvinces'); map.getPane('paneProvinces').style.zIndex = 401;
map.createPane('paneFarms');     map.getPane('paneFarms').style.zIndex = 402;
map.createPane('paneParcels');   map.getPane('paneParcels').style.zIndex = 403;
map.createPane('paneServitudes');map.getPane('paneServitudes').style.zIndex = 404;
map.createPane('paneSelection'); map.getPane('paneSelection').style.zIndex = 405;
map.createPane('paneKml');       map.getPane('paneKml').style.zIndex = 406;
map.createPane('paneLabels');    map.getPane('paneLabels').style.zIndex = 407;
```

**Gotcha hit here**: a refactor once left behind a reference to a never-defined
`CAT_PANE` lookup object after the categories were restructured — a silent
`ReferenceError` that made *every* layer fail to render with no obvious error
surfaced to the user. If layers vanish entirely (not just one), check for stale
references to renamed/removed lookup tables before assuming a data problem.

---

## 3. Scale-gated "secondary" layers

Requirement: only Province is on by default; a fixed set of detail layers
(Erven, Holding, Public Place, Township, Farm Portion, Servitude Area,
Servitude Line) should auto-enable once the user has zoomed in past a
readable scale, and auto-disable again on zooming back out.

**Key lesson — use the displayed scale bar text, not raw zoom level.** Zoom
level ↔ real-world scale isn't 1:1 (it depends on latitude), and the user's
own mental model was "the number on the scale bar in the bottom-left", not the
Leaflet zoom integer. Read the rendered scale control directly:

```js
function getDisplayedScaleMeters(){
  const el = document.querySelector('.leaflet-control-scale-line');
  if(!el) return null;
  const txt = el.textContent.trim(); // e.g. "100 m" or "1.5 km"
  const m = txt.match(/^([\d.]+)\s*(m|km)$/);
  if(!m) return null;
  return parseFloat(m[1]) * (m[2] === 'km' ? 1000 : 1);
}
```

**Key lesson — react to threshold *crossings*, not levels.** An early version
recomputed "should secondary layers be on?" from scratch on every `zoomend`
and just set checkboxes accordingly — this correctly turned layers *on* when
zooming in, but on zooming back out the same “desired state” logic never fired
a transition, so layers stayed on. Track the crossing explicitly:

```js
const SECONDARY_AUTO_SCALE_METERS = 100;
let secondaryLayersOn = false; // tracks the *previous* state, not derived fresh each time

function applySecondaryScaleGate(){
  const scaleM = getDisplayedScaleMeters();
  if(scaleM == null) return;
  const shouldBeOn = scaleM <= SECONDARY_AUTO_SCALE_METERS;
  if(shouldBeOn === secondaryLayersOn) return; // no crossing — do nothing
  secondaryLayersOn = shouldBeOn;
  LAYER_DEFS.filter(l => l.secondary).forEach(l => { l.checked = shouldBeOn; syncLayer(l); });
  renderLegend();
}
// IMPORTANT: bind to 'moveend', not 'zoomend' — Leaflet's own scale control
// updates its DOM text on 'moveend', so reading it from a 'zoomend' handler
// can race and read the stale pre-update text.
map.on('moveend', applySecondaryScaleGate);
```

**Key lesson — don't persist auto-managed state to localStorage.** Secondary
layers' checked state is *derived* from current scale, not a user preference.
If you persist it and restore it on reload, you get a broken hybrid: the
restored `checked=true` doesn't match `secondaryLayersOn=false` (which always
resets to `false` on load), so the very first scale-crossing check sees "no
change" and never corrects the mismatch. Fix: explicitly exclude these keys
from both saving and restoring.

```js
function persistChecked(){
  const state = {};
  LAYER_DEFS.forEach(l => { if(!l.secondary) state[l.key] = l.checked; });
  localStorage.setItem('layerChecked', JSON.stringify(state));
}
// on restore: force l.checked = l.secondary ? false : (saved value ?? l.defaultChecked)
```

---

## 4. Geolocation search, matched to a companion app's API

When told "use the same API as our other app (`beacons.html`)", the fastest
reliable approach was reading the *other file* directly for its exact
Mapbox Geocoding endpoint, token, and debounce timing, rather than guessing
at a "reasonable" implementation — mismatched debounce/endpoint parameters are
the kind of thing that looks fine in testing but behaves subtly differently
under real use.

**Key lesson — match zoom-after-search to the same scale-bar semantics as
section 3.** "Zoom to the search result at a useful scale" should produce the
*same* scale-bar reading the secondary-layer gate uses (100 m), not just "zoom
level 17" — because at different latitudes, the same zoom level produces a
different real-world scale (Leaflet's scale bar accounts for latitude;
zoom-level does not). Replicate Leaflet's own scale math to solve for the
zoom that produces a target scale-bar reading at a given latitude:

```js
function leafletNiceScaleNumber(num){
  // Mirrors L.Control.Scale._getRoundNum: rounds to the "nice" 1/2/3/5×10^n
  // step the scale bar itself would choose, so the target matches exactly
  // what the control will end up displaying.
  const pow10 = Math.pow(10, Math.floor(Math.log10(num)));
  let d = num / pow10;
  d = d >= 10 ? 10 : d >= 5 ? 5 : d >= 3 ? 3 : d >= 2 ? 2 : 1;
  return pow10 * d;
}
function zoomForScaleMeters(lat, targetMeters){
  const maxWidthPx = 160; // Leaflet's default L.Control.Scale maxWidth
  // metersPerPixel at zoom z, latitude lat:
  //   metersPerPixel = 156543.03392 * cos(lat*pi/180) / 2^z
  // solve for z such that leafletNiceScaleNumber(metersPerPixel*maxWidth) == targetMeters
  for(let z = 22; z >= 0; z--){
    const mpp = 156543.03392 * Math.cos(lat * Math.PI/180) / Math.pow(2, z);
    if(leafletNiceScaleNumber(mpp * maxWidthPx) <= targetMeters) return z;
  }
  return 0;
}
```

---

## 5. Multi-layer feature-click popups

Requirement: clicking a point where several layers' polygons/lines overlap
should show *all* of them, with prev/next navigation — not just the topmost
one Leaflet would naturally hit-test.

**Approach**: don't rely on Leaflet's own per-layer click event (which only
fires for the topmost feature under the cursor). Instead, on map click,
manually hit-test every currently-visible layer's GeoJSON against the click
point:

```js
function hitTestLayer(l, latlng, pt){
  if(!l.instance || !map.hasLayer(l.instance)) return [];
  const hits = [];
  l.instance.eachLayer(feLayer => {
    const f = feLayer.feature;
    if(!f || !f.geometry) return;
    if(f.geometry.type === 'Polygon' || f.geometry.type === 'MultiPolygon'){
      if(turf.booleanPointInPolygon([latlng.lng, latlng.lat], f)) hits.push({ layerDef:l, feature:f });
    } else if(f.geometry.type === 'LineString' || f.geometry.type === 'MultiLineString'){
      // Polylines have no interior — use a pixel-space distance tolerance
      // instead of an exact geometric test, matching what a mouse click
      // actually feels precise enough to mean.
      const distKm = minDistanceToLineGeometryKm(f.geometry, latlng);
      const pxTolerance = 6;
      const metersPerPixel = 40075016.686 * Math.abs(Math.cos(latlng.lat*Math.PI/180)) / (256*Math.pow(2, map.getZoom()));
      if(distKm * 1000 <= pxTolerance * metersPerPixel) hits.push({ layerDef:l, feature:f });
    }
  });
  return hits;
}
```

Then aggregate hits across every checked layer, build one popup with a
prev/next index (`activeHits`, `activeHitIndex`), and re-render its content in
place rather than opening a new popup per click-through.

**Gotcha hit here — horizontal scrollbar appeared even though nothing
overflowed horizontally.** Root cause: CSS's `overflow` shorthand auto-computes
`overflow-x` to `auto` whenever `overflow-y` is set to a non-`visible` value
and `overflow-x` is left unset — a spec quirk, not a sizing bug. Fix is to be
explicit:

```css
.leaflet-popup-content { overflow-x: hidden !important; overflow-y: auto; }
```

**Design lesson**: put the prev/next controls as visually obvious side arrows
on the popup shell itself (not, say, small icons buried in a corner) — "the
user should be able to tell at a glance that there's more than one layer
here" was an explicit requirement, and it's an easy thing to under-deliver on
if you only think about *functional* correctness.

---

## 6. Area of Interest (AOI) system

AOI = a user-supplied boundary (KML/KMZ upload or hand-drawn polygon) that
the app intersects against every layer on the live ArcGIS service, then
shows only the intersecting subset.

**Key lesson — general/browse layers and AOI-result layers are mutually
exclusive, and this needs an explicit gate, not an implicit one.** The first
attempt at "hide the general layers while an AOI is active" didn't add any
such gate at all — it just assumed that loading AOI results would naturally
crowd out the general view. It didn't; both rendered simultaneously, which
was confusing. The fix is a single boolean checked everywhere layers decide
whether to render:

```js
let aoiActive = false; // true whenever an AOI is loaded (KML/KMZ or hand-drawn)

function syncLayer(l){
  const shouldShow = l.checked && isAboveScale(l) && !aoiActive;
  if(shouldShow){
    if(!l.instance) l.instance = createInstance(l);
    if(!map.hasLayer(l.instance)) l.instance.addTo(map);
    LabelEngine.register(l, l.instance);
  } else {
    if(l.instance && map.hasLayer(l.instance)) map.removeLayer(l.instance);
    LabelEngine.unregister(l.key);
  }
}
```

`aoiActive` is set `true` at the start of loading an AOI and reset to `false`
in `clearAOI()` (which then re-runs `syncAllLayers()` to restore the general
view). **Lesson for debugging this class of bug**: when two things "should be
mutually exclusive" but aren't, look for a missing *shared gate* before
assuming a per-feature logic bug — the fix here was one added condition
(`&& !aoiActive`), not a rewrite of either code path.

**Design decisions worth keeping**:
- AOI result layers are sorted by feature count (most features first) so the
  panel surfaces the layers with the most relevant hits at the top.
- Province is excluded from AOI *results being pre-checked* by default (it's
  usually just administrative context, not something people want to export).
- The AOI layers list in the panel is collapsible — this mattered because the
  download-format buttons need to be visible immediately below "Area", not
  pushed off-screen by a long, always-expanded layer list.

---

## 7. Label Engine (on-map text labels)

Requirement: parcels/farms/etc. show their `TAG_VALUE` (or `PROVINCE` for the
Province layer) as floating text at the feature's tag anchor point
(`TAG_X`/`TAG_Y` fields from the ArcGIS service — these already encode where
the *original* cartographer placed the label, so use them instead of
recomputing a centroid).

```js
function labelAnchorForFeature(feature){
  const p = feature.properties || {};
  if(typeof p.TAG_X === 'number' && typeof p.TAG_Y === 'number'
     && isFinite(p.TAG_X) && isFinite(p.TAG_Y) && (p.TAG_X || p.TAG_Y)){
    return [p.TAG_X, p.TAG_Y];
  }
  try{ return turf.pointOnFeature(feature).geometry.coordinates; }catch(e){ return null; }
}
```

A single `LabelEngine` object owns one Leaflet layer group for *all* labels
(not one per data layer), with explicit `register(layerDef, esriLayer)` /
`unregister(key)` calls and a debounced `refresh()`:

```js
const LabelEngine = {
  group: L.layerGroup([], { pane:'paneLabels' }),
  sources: new Map(),
  register(layerDef, esriLayer){ this.sources.set(layerDef.key, { layerDef, esriLayer }); this.scheduleRefresh(); },
  unregister(key){ this.sources.delete(key); this.scheduleRefresh(); },
  scheduleRefresh(){ clearTimeout(this._t); this._t = setTimeout(() => this.refresh(), 150); },
  refresh(){ /* rebuild all label markers from this.sources */ }
};
```

**Lesson — general layers and AOI-result layers must call register/unregister
at every actual state-change point, not rely on a "live getter" that reaches
into current app state when refresh() runs.** A live-getter approach is
tempting (less code to keep in sync) but is fragile against ordering bugs
that are hard to prove correct by static reading alone. When an AOI-label bug
couldn't be conclusively root-caused by inspection, the defensible fix was to
refactor to the same explicit-call pattern already proven correct for general
layers — matching a known-good pattern beats trying to out-clever a
hard-to-verify one, especially under time pressure.

---

## 8. Export pipeline: shared symbology

Every export format embeds the *same* colours/weights/dash patterns used on
the map, so styling isn't lost the moment data leaves the app. Two small
shared helpers make every format-specific function trivial:

```js
function colorToRgbArray(hex){
  hex = String(hex).replace('#','');
  return [parseInt(hex.slice(0,2),16), parseInt(hex.slice(2,4),16), parseInt(hex.slice(4,6),16)];
}
function kmlColor(hex, alphaFraction){
  // KML colour order is aabbggrr (alpha first, then blue/green/red — the
  // REVERSE of CSS #rrggbb). Easy to get backwards; verify with a known
  // color once (e.g. #e2231a -> alpha=ff, blue=1a, green=23, red=e2 -> "ff1a23e2").
  const [r,g,b] = colorToRgbArray(hex);
  const a = Math.round((alphaFraction ?? 1) * 255);
  const h = n => Math.max(0, Math.min(255, Math.round(n))).toString(16).padStart(2,'0');
  return h(a) + h(b) + h(g) + h(r);
}
```

---

## 9. KML/KMZ export

### 9.1 Style blocks need explicit `<color>`/`<outline>`, not just `<fill>0</fill>`

A bare `<PolyStyle><fill>0</fill></PolyStyle>` (no `<color>`, no `<outline>`)
is silently ignored by some KML viewers, which then fall back to an **opaque
white fill** — the opposite of what was intended. Always be fully explicit:

```js
function kmlStyleBlockFor(layerDef, styleId){
  const isLine = layerDef.type === 'polyline';
  const lineColor = kmlColor(layerDef.color, 1);
  const fillAlpha = isLine ? 0 : (layerDef.fillOpacity ?? 0);
  const fillTag = isLine ? '' : `<PolyStyle><color>${kmlColor(layerDef.color, fillAlpha)}</color><fill>${fillAlpha > 0 ? 1 : 0}</fill><outline>1</outline></PolyStyle>`;
  return `<Style id="${styleId}"><LineStyle><color>${lineColor}</color><width>${layerDef.weight}</width></LineStyle>${fillTag}</Style>`;
}
```

### 9.2 Replicating an on-map "halo" effect (AOI boundary)

The on-map AOI boundary uses two stacked Leaflet layers — a wide, low-opacity
glow plus a crisp solid core line:

```js
// On-map (Leaflet):
aoiLayer = L.layerGroup([
  L.geoJSON(geojson, { style:{ color:'#ff2b2b', weight:11, opacity:.22, fill:false } }),
  L.geoJSON(geojson, { style:{ color:'#e2231a', weight:4.5, opacity:1, fill:false } })
]).addTo(map);
```

The KML equivalent needs *two separate styled placemark sets* (KML has no
concept of a single line with two rendered passes) — reuse the same explicit
color+fill+outline pattern as above, once per "layer" of the halo:

```js
const aoiHaloStyle = `<Style id="aoiHaloStyle"><LineStyle><color>${kmlColor('#ff2b2b', .22)}</color><width>11</width></LineStyle><PolyStyle><color>${kmlColor('#ff2b2b', 0)}</color><fill>0</fill><outline>1</outline></PolyStyle></Style>`;
const aoiCoreStyle = `<Style id="aoiCoreStyle"><LineStyle><color>${kmlColor('#e2231a', 1)}</color><width>4.5</width></LineStyle><PolyStyle><color>${kmlColor('#e2231a', 0)}</color><fill>0</fill><outline>1</outline></PolyStyle></Style>`;
const aoiFolder = `<Folder><name>Area of Interest</name>${aoiHaloStyle}${aoiCoreStyle}
  <Folder><name>Glow</name>${features.map((f,i)=>featureToPlacemark(f,'AOI',i,'aoiHaloStyle')).join('')}</Folder>
  <Folder><name>Boundary</name>${features.map((f,i)=>featureToPlacemark(f,'AOI',i,'aoiCoreStyle')).join('')}</Folder>
</Folder>`;
```

### 9.3 Text labels in KML: hidden-icon point placemarks

A polygon Placemark's own `<name>` isn't reliably rendered as a floating
label by every KML viewer. The reliable, widely-supported technique (matched
against a real reference export from another survey tool) is a **separate
Point Placemark per label**, with its icon hidden via `IconStyle scale=0`:

```js
function kmlLabelStyleBlockFor(layerDef, styleId){
  return `<Style id="${styleId}"><IconStyle><scale>0</scale></IconStyle><LabelStyle><color>${kmlColor(layerDef.color,1)}</color><scale>0.9</scale></LabelStyle></Style>`;
}
function labelPointPlacemarksFor(layerResult, styleId){
  if(!layerResult.layerDef.labelField) return '';
  return layerResult.geojson.features.map(f => {
    const text = (f.properties || {})[layerResult.layerDef.labelField];
    if(!text || !f.geometry) return '';
    const anchor = labelAnchorForFeature(f); // see section 7
    if(!anchor) return '';
    return `<Placemark><name>${xmlEsc(text)}</name><styleUrl>#${styleId}</styleUrl><Point><coordinates>${anchor[0]},${anchor[1]},0</coordinates></Point></Placemark>`;
  }).join('');
}
```
Put these in their own `"<Layer Name> Labels"` folder per layer so they can be
toggled independently of the boundary geometry in the KML viewer's layer tree.

---

## 10. Shapefile export

Built by hand (no library) — DataView-based `.shp`/`.shx`, and a hand-rolled
`.dbf`:

- **Field names are truncated to 10 characters and upper-cased** (the DBF
  format's hard limit) — collisions after truncation are resolved by
  appending a numeric suffix that still fits in 10 characters:
  ```js
  let name = k.toUpperCase().replace(/[^A-Z0-9_]/g,'_').slice(0,10) || 'FIELD';
  let base = name, n = 1;
  while(used.has(name)){ name = base.slice(0, 10-String(n).length) + n; n++; }
  ```
- **Polygon ring winding matters for Esri Shapefile compatibility**: outer
  rings must be clockwise, holes counter-clockwise — the opposite convention
  from GeoJSON's right-hand rule. Normalize explicitly per ring rather than
  assuming the source data already matches.
- A `.qml` sidecar (see section 11) is zipped alongside `.shp/.shx/.dbf/.prj`
  so QGIS picks up the app's symbology automatically via "Load Style" (this
  works because QGIS looks for a same-named `.qml` next to a shapefile
  automatically on load, in addition to the manual load-style menu).

---

## 11. QGIS `.qml` style files — the schema lessons that mattered most

This was the single most error-prone part of the whole project, because the
QML schema is undocumented in most public references and its structure
doesn't match how you'd naively design it. **These lessons came from reading
QGIS's actual C++ source** (`qgspallabeling.cpp`, `qgsvectorlayerlabeling.cpp`,
`qgstextformat.cpp` on GitHub) after speculative fixes based on "how it looks
like it should work" repeatedly failed — when a style/schema format is
under-documented, go to the source that actually reads it, not another
guess.

### 11.1 `fieldName`/`isExpression` are ATTRIBUTES of `<text-style>`, not their own elements

**The bug**: an earlier version emitted the label's source field as
standalone sibling elements:
```xml
<!-- WRONG — QGIS's reader never looks for these -->
<labeling type="simple">
  <settings calloutType="simple">...</settings>
  <fieldName>TAG_VALUE</fieldName>
  <isExpression>0</isExpression>
</labeling>
```
QGIS's actual reader (`QgsPalLayerSettings::readXml`) does this:
```cpp
QDomElement textStyleElem = elem.firstChildElement( "text-style" );
fieldName = textStyleElem.attribute( "fieldName" );      // an XML ATTRIBUTE
isExpression = textStyleElem.attribute( "isExpression" ).toInt();
```
So the field name has to be an **attribute on `<text-style>`**, nested inside
`<settings>`, nested inside `<labeling>`:
```xml
<labeling type="simple">
  <settings calloutType="simple">
    <text-style fieldName="TAG_VALUE" isExpression="0" fontFamily="Arial" fontSize="9" fontSizeUnit="Point" textColor="0,255,0,255" textOpacity="1">
      <text-buffer bufferDraw="1" bufferSize="1" bufferSizeUnits="MM" bufferColor="0,0,0,255" bufferOpacity="1" bufferNoFill="1"/>
    </text-style>
    <text-format wrapChar="" autoWrapLength="0" multilineAlign="0"/>
    <placement placement="0" quadOffset="4" xOffset="0" yOffset="0" rotationAngle="0" fitInPolygonOnly="1" dist="0" overlapHandling="AllowOverlapAtNoCost"/>
    <rendering drawLabels="1" scaleVisibility="0" upsidedownLabels="0" obstacle="0"/>
  </settings>
</labeling>
```
With the field name in the wrong place, QGIS reads `fieldName` back as an
empty string, `prepare()` then returns `false` for that layer's labeling
entirely (see `QgsPalLayerSettings::prepare()`:
`if (fieldName.isEmpty()) return false;`), and QGIS silently falls back to
whatever field its own "friendly display field" heuristic guesses — which,
in this dataset, happened to land on a field containing "NAME" in it
(`SS_NAME`), producing the very confusing symptom of *"labels show the wrong
field"* rather than *"labels don't show at all"*.

**How to verify a schema guess like this without a live QGIS instance**:
fetch the actual reader/writer source for the format in question
(`raw.githubusercontent.com/<project>/<repo>/master/<path>`), and trace the
exact `element.attribute(...)` / `element.firstChildElement(...)` calls —
this tells you definitively where a value has to live, rather than inferring
schema shape from example files (which may themselves be from a different
version) or from guessing based on naming conventions.

### 11.2 `previewExpression`/`displayfield` govern the *identify/display* name, separately from labeling

Even after fixing the on-map label field, QGIS's Identify tool / feature list
still shows its own auto-guessed "friendly" field unless you *also* pin the
layer's display name explicitly:
```xml
<qgis version="3.28" styleCategories="Symbology|Labeling|LayerConfiguration">
  ...
  <previewExpression>"TAG_VALUE"</previewExpression>
  <displayfield>TAG_VALUE</displayfield>
</qgis>
```
Note the `LayerConfiguration` category has to be added to `styleCategories`
too, or QGIS's style loader won't even attempt to apply these.

### 11.3 `obstacle="1"` is QGIS's SILENT DEFAULT, and it breaks multi-layer setups where layers geographically nest

**The bug**: "each layer's labels show fine alone, but disappear the moment a
second layer is turned on." Root cause, again found by reading the actual
reader:
```cpp
mObstacleSettings.setIsObstacle( renderingElem.attribute( "obstacle", "1" ).toInt() );
```
The default (used whenever the attribute is *absent*, which every earlier
version of this code did) is `"1"` — every polygon layer, by default, blocks
*every other layer's* labels from being placed anywhere over its own
footprint. Cadastral layers nest geographically (Erven sit inside Farm
Portion sit inside Township), so turning on a second, larger overlapping
layer silently wipes out the first layer's labels across its entire extent.
The live web app has no such cross-layer collision logic at all (its
`LabelEngine` always draws every active layer's labels unconditionally), so
matching that behavior in the export means explicitly opting out of QGIS's
default:
```xml
<rendering drawLabels="1" ... obstacle="0"/>
<placement ... overlapHandling="AllowOverlapAtNoCost"/>
```
`overlapHandling="AllowOverlapAtNoCost"` additionally stops QGIS suppressing
a label because it visually collides with *another label* (as opposed to
another layer's geometry, which `obstacle` covers) — both were needed to
fully replicate "always show every active layer's labels."

**General lesson**: when a rendering engine's behavior changes based on
*combinations* of layers/settings rather than a single layer in isolation,
suspect a **default value that only shows an effect once something else is
also present** — test hypotheses by asking "what's different about the
one-layer case?" rather than only "what's wrong with this one layer's
config?".

### 11.4 Full worked `buildQmlLabeling()` / `buildQml()`

```js
function buildQmlLabeling(layerDef){
  if(!layerDef.labelField) return { categories:'', xml:'<labeling-enabled>0</labeling-enabled>', previewExpr:'' };
  const [r,g,b] = colorToRgbArray(layerDef.color);
  const fieldName = xmlEsc(layerDef.labelField);
  const xml = `<labeling-enabled>1</labeling-enabled>
  <labeling type="simple">
    <settings calloutType="simple">
      <text-style fieldName="${fieldName}" isExpression="0" useSubstitutions="0" legendString="Aa" fontFamily="Arial" fontWeight="${layerDef.labelBold ? 75 : 50}" fontBold="${layerDef.labelBold ? 1 : 0}" fontItalic="0" fontStrikeout="0" fontUnderline="0" fontSize="9" fontSizeUnit="Point" textColor="${r},${g},${b},255" textOpacity="1">
        <text-buffer bufferDraw="1" bufferSize="1" bufferSizeUnits="MM" bufferColor="0,0,0,255" bufferOpacity="1" bufferNoFill="1"/>
      </text-style>
      <text-format wrapChar="" autoWrapLength="0" multilineAlign="0"/>
      <placement placement="${layerDef.type === 'polyline' ? 2 : 0}" quadOffset="4" xOffset="0" yOffset="0" rotationAngle="0" fitInPolygonOnly="1" dist="0" overlapHandling="AllowOverlapAtNoCost"/>
      <rendering drawLabels="1" scaleVisibility="0" upsidedownLabels="0" fontLimitPixelSize="0" fontMinPixelSize="3" fontMaxPixelSize="10000" obstacle="0"/>
    </settings>
  </labeling>`;
  return { categories:'|Labeling|LayerConfiguration', xml, previewExpr: layerDef.labelField };
}

function buildQml(layerDef){
  const [r,g,b] = colorToRgbArray(layerDef.color);
  const isLine = layerDef.type === 'polyline';
  const penStyle = layerDef.dash ? 'dash' : 'solid';
  const outlineWidthMm = ((layerDef.weight || 1) * 0.28).toFixed(2);
  const fillAlpha255 = Math.round((layerDef.fillOpacity ?? 0) * 255);
  const symbolXml = isLine
    ? `<symbol type="line" alpha="1" clip_to_extent="1" name="0" force_rhr="0">
        <layer pass="0" class="SimpleLine" locked="0" enabled="1">
          <Option type="Map">
            <Option type="QString" name="line_color" value="${r},${g},${b},255"/>
            <Option type="QString" name="line_style" value="${penStyle}"/>
            <Option type="QString" name="line_width" value="${outlineWidthMm}"/>
            <Option type="QString" name="line_width_unit" value="MM"/>
            <Option type="QString" name="capstyle" value="square"/>
          </Option>
        </layer>
      </symbol>`
    : `<symbol type="fill" alpha="1" clip_to_extent="1" name="0" force_rhr="0">
        <layer pass="0" class="SimpleFill" locked="0" enabled="1">
          <Option type="Map">
            <Option type="QString" name="color" value="${r},${g},${b},${fillAlpha255}"/>
            <Option type="QString" name="outline_color" value="${r},${g},${b},255"/>
            <Option type="QString" name="outline_style" value="${penStyle}"/>
            <Option type="QString" name="outline_width" value="${outlineWidthMm}"/>
            <Option type="QString" name="outline_width_unit" value="MM"/>
            <Option type="QString" name="style" value="solid"/>
          </Option>
        </layer>
      </symbol>`;
  const labeling = buildQmlLabeling(layerDef);
  const previewBlock = labeling.previewExpr
    ? `\n  <previewExpression>"${xmlEsc(labeling.previewExpr)}"</previewExpression>\n  <displayfield>${xmlEsc(labeling.previewExpr)}</displayfield>`
    : '';
  return `<!DOCTYPE qgis PUBLIC 'http://mrcc.com/qgis.dtd' 'SYSTEM'>
<qgis version="3.28" styleCategories="Symbology${labeling.categories}">
  <renderer-v2 type="singleSymbol" forceraster="0" symbollevels="0" referencescale="-1" enableorderby="0">
    <symbols>
      ${symbolXml}
    </symbols>
  </renderer-v2>
  ${labeling.xml}${previewBlock}
</qgis>`;
}
```

---

## 12. GeoPackage export (via sql.js)

Built as a hand-rolled SQLite database (loaded lazily via `sql.js`, not
bundled up-front, since most sessions never touch this export format):

- Required GeoPackage core tables: `gpkg_spatial_ref_sys`, `gpkg_contents`,
  `gpkg_geometry_columns`.
- **`layer_styles` is the actual mechanism that carries symbology into
  QGIS** — a QGIS-standard table name/schema that QGIS auto-detects and
  loads styling from when opening a `.gpkg`. This — *not* a `.gpkg.aux.xml`
  sidecar — is the correct place for style data:
  ```sql
  CREATE TABLE layer_styles (
    id INTEGER PRIMARY KEY AUTOINCREMENT, f_table_catalog TEXT, f_table_schema TEXT,
    f_table_name TEXT, f_geometry_column TEXT, styleName TEXT, styleQML TEXT,
    styleSLD TEXT, useAsDefault BOOLEAN, description TEXT, owner TEXT, ui TEXT,
    update_time DATETIME
  );
  ```
  Each row's `styleQML` column is exactly the same `buildQml(layerDef)`
  output used for the Shapefile `.qml` sidecar — one function serving two
  export formats.

### 12.1 Lesson: `.gpkg.aux.xml` is a red herring for vector styling

A user request initially asked for a `.gpkg.aux.xml` sidecar "to maintain
symbology." **This doesn't work**: `.aux.xml` is GDAL's PAM (Persistent
Auxiliary Metadata) mechanism, used for *raster* band statistics/colour
tables — there's no vector-symbology schema for it, and no GIS software
(QGIS, ArcGIS, GDAL/OGR itself) reads one back as layer styling for a
GeoPackage. If asked for this, it's worth flagging the technical caveat
(as was done here) even while implementing what's asked, so the person isn't
left thinking the file did something it didn't — and worth reverting cleanly
when they come back around to the same conclusion, rather than leaving dead
code that only pretends to help.

### 12.2 Every export format that includes derived/synthetic geometry (like an AOI boundary) needs its own explicit code path

The AOI boundary polygon isn't one of the live ArcGIS layers — it's
synthesized client-side from an upload or hand-drawing. It's easy to build an
export pipeline that only iterates "the checked ArcGIS-sourced layers" and
forget this one, non-layer-catalogue geometry entirely (which is exactly
what happened here — KML got an AOI folder from the start, but GeoPackage
was only fixed once explicitly reported as missing it). **Checklist for any
export format that has multiple geometry sources**: enumerate every source
type (live layers, AOI boundary, any hand-drawn annotations) explicitly, not
just "the collection this format naturally iterates."

```js
// Shared helper: creates one feature table + matching layer_styles row.
// Used both for real ArcGIS-sourced layers AND the synthetic AOI boundary.
function writeGpkgLayer(db, table, layerDefLike, features, styleLabel){
  const fieldDefs = buildSqlFieldDefs(features);
  const colsSql = fieldDefs.map(f => `"${f.name}" ${f.sqlType}`).join(', ');
  db.run(`CREATE TABLE "${table}" (fid INTEGER PRIMARY KEY AUTOINCREMENT, geom BLOB${colsSql ? ', '+colsSql : ''});`);
  // ...insert features as WKB geometry blobs + attribute columns...
  db.run(`INSERT INTO layer_styles (...) VALUES (..., ?)`, [buildQml(layerDefLike)]);
}

// Real layers:
checked.forEach(r => writeGpkgLayer(db, sanitizeSqlIdent(r.layerDef.name), r.layerDef, r.geojson.features, r.layerDef.name + ' (style)'));

// AOI boundary — a layerDef-like object built on the fly, matching on-map styling:
if(aoiGeojson && aoiGeojson.features.length){
  const aoiLayerDefLike = { name:'Area of Interest', type:'polygon', color:'#e2231a', weight:4.5, fillOpacity:0 };
  writeGpkgLayer(db, sanitizeSqlIdent(aoiLayerDefLike.name), aoiLayerDefLike, aoiGeojson.features, 'Area of Interest (style)');
}
```

---

## 13. Attribute summary CSV — a plain-language export alongside every format

Requirement: alongside every download format (KML/KMZ/Shapefile/GeoPackage),
also produce a small CSV with just the handful of fields people actually
need per layer type, aliased into plain language — critically, the
21-digit LPI/SG code used afterwards to bulk-download parcel diagrams from a
separate system.

### 13.1 Field-group-per-layer-type config, not per-field logic

```js
const CSV_FIELD_GROUPS = {
  erven:       { fields:['TAG_VALUE','MIN_REGION','ID'], aliases:['Property','Township','21 Digit LPI Code'] },
  holding:     { fields:['TAG_VALUE','MIN_REGION','ID'], aliases:['Property','Township','21 Digit LPI Code'] },
  publicplace: { fields:['TAG_VALUE','MIN_REGION','ID'], aliases:['Property','Township','21 Digit LPI Code'] },
  farmportion: { fields:['TAG_VALUE','FARM_NAME','ID'], aliases:['Property','Farm Name','21 Digit LPI Code'] },
  parentfarm:  { fields:['TAG_VALUE','FARM_NAME','ID'], aliases:['Property','Farm Name','21 Digit LPI Code'] },
  servarea:    { fields:['TAG_VALUE'], aliases:['S.G. Number'] },
  servline:    { fields:['TAG_VALUE'], aliases:['S.G. Number'] },
  leasearea:   { fields:['TAG_VALUE'], aliases:['S.G. Number'] },
};
// Fixed reading order — independent of whatever sort order the AOI results
// panel uses (there, layers are sorted by feature count) — so the CSV always
// lists layers the same way regardless of which area was loaded.
const CSV_LAYER_ORDER = ['erven','holding','publicplace','farmportion','parentfarm','servarea','servline','leasearea'];
```

**Lesson**: when the ArcGIS service's field schema doesn't match assumptions
(e.g. an initial guess used `MAJ_REGION` for "Farm Name" before being
corrected to the actual dedicated `FARM_NAME` field), fetching the live
service's layer metadata (`.../MapServer/<id>?f=json`) and reading its
`fields` array directly settles the question — field *aliases* returned by
the service (e.g. `"alias":"Farm Name"` for `FARM_NAME`) are a strong, ground
-truth signal for which field a person means when they name it in plain
language.

### 13.2 Sectioned layout, one section per layer

```js
function buildAttributesCsv(getCheckedAoiResults){
  const checked = getCheckedAoiResults();
  const byKey = new Map(checked.map(r => [r.layerDef.key, r]));
  const sections = [];
  CSV_LAYER_ORDER.forEach(key => {
    const group = CSV_FIELD_GROUPS[key];
    const r = byKey.get(key);
    if(!r || !r.geojson.features.length) return; // skip layers not checked / with no hits
    const lines = [r.layerDef.name, group.aliases.map(csvEscape).join(',')];
    r.geojson.features.forEach(f => {
      const props = f.properties || {};
      lines.push(group.fields.map(fld => csvEscape(fld === 'ID' ? csvForceText(props[fld]) : props[fld])).join(','));
    });
    sections.push(lines.join('\r\n'));
  });
  return sections.length ? sections.join('\r\n\r\n') + '\r\n' : null; // null => caller skips the download
}
```

Each section is: a title line (plain layer name), a header line (the
aliases), one row per feature, then a blank line before the next section.
Returning `null` when nothing qualifies lets the caller skip the download
entirely rather than shipping a confusing empty file.

### 13.3 CSV cannot carry cell formatting — know what's actually achievable

When asked to make "everything left-aligned and layer titles bold" in the
CSV: **a plain CSV file has no styling metadata at all** — no bold, no
alignment, no cell colors. Those are spreadsheet-*file* (e.g. `.xlsx`
`styles.xml`) features, not CSV. Rather than silently doing something
partial and letting the person discover the limitation later, it's worth
surfacing the constraint and the realistic options directly (plain CSV
best-effort vs. switching to a real `.xlsx`), and being explicit that
"titles bold" specifically **cannot** be approximated by a text hack like
ALL-CAPS or `**stars**` without also silently changing the literal text that
downstream scripts might string-match against.

What *is* achievable, and worth doing regardless of which option someone
picks: Excel auto-detects any long, all-digit string as a *number*, which
(a) right-aligns it — the actual "alignment" symptom people usually notice —
and (b) far more seriously, **silently rounds it to ~15 significant digits**,
corrupting exactly the kind of long identifier code (a 21-digit LPI code
here) the file exists to carry. The standard fix is wrapping the value as an
Excel formula-literal so Excel evaluates it as text instead of parsing it as
a number:

```js
function csvForceText(v){
  if(v === null || v === undefined || v === '') return '';
  return '="' + String(v).replace(/"/g,'""') + '"';
}
function csvEscape(v){
  if(v === null || v === undefined) return '';
  const s = String(v);
  return /[",\r\n]/.test(s) ? '"' + s.replace(/"/g,'""') + '"' : s;
}
```

`csvForceText('021003000000012300000')` → `="021003000000012300000"`, which
`csvEscape` then correctly double-quotes (since it now contains `"`
characters) to `"=""021003000000012300000"""` in the raw file. Excel opens
this and evaluates it as a formula returning literal text — left-aligned,
digits intact, no scientific notation. **Trade-off to flag explicitly**: a
script parsing the raw CSV text (not opening it in Excel) will see the
`="..."` wrapper too, and needs one extra strip step
(`value.replace(/^="|"$/g, '')`) to recover the bare code. This is exactly
the kind of trade-off worth surfacing to the person rather than silently
picking a side — formatting-for-humans and clean-value-for-scripts are in
real tension here, and only they know which matters more for their
downstream automation.

### 13.4 Wire it into every export function, and skip silently when there's nothing to write

```js
function downloadAttributesCsv(){
  const csv = buildAttributesCsv(getCheckedAoiResults);
  if(!csv) return;
  downloadBlob(new Blob([csv], { type:'text/csv;charset=utf-8' }), 'csg_cadastral_attributes.csv');
}
// Call once at the end of each export function, after its own main download:
//   exportKML()        -> downloadAttributesCsv();
//   exportShapefile()  -> downloadAttributesCsv();
//   exportGeoPackage() -> downloadAttributesCsv();
```

---

## 14. Verification methodology (repeat after every change)

Since this whole app is a single inline `<script>`/`<style>` block with no
build step or test runner, the same lightweight verification pass was run
after nearly every change — worth replicating in any similar single-file
project:

1. **Syntax check the extracted script**: pull everything between
   `<script>`/`</script>` into a standalone `.js` file and run
   `node --check` on it. Catches typos/mismatched brackets immediately,
   before ever loading the page.
2. **HTML tag-balance check**: a small regex-based stack parser over the
   *markup only* (with `<script>`/`<style>` bodies blanked out first, so
   template-literal HTML strings inside JS don't get mistaken for real page
   markup) — catches unclosed/mismatched tags from a bad edit.
3. **`getElementById` cross-reference**: collect every static `id="..."` in
   the markup and every `getElementById('...')` call in the script, and
   diff them — catches renamed/typo'd IDs. **Caveat**: IDs generated inside
   JS template literals (e.g. popup HTML built at runtime) will show as
   "missing" by this check even though they're fine — expect and allow for
   that category of false positive rather than chasing it.
4. **Duplicate-ID scan restricted to markup *before* the first
   `<script>` tag**: scanning the *whole file* for duplicate `id="..."`
   produces false positives from JS template-literal strings that
   legitimately repeat the same ID pattern in different generated-HTML
   functions (e.g. `id="${styleId}"` appearing in two different KML-folder
   builders). Restricting the scan to `html.split('<script>')[0]` avoids
   this entirely.
5. **CSS brace-balance check**: a simple counter over the `<style>` block
   content.
6. **For generated XML output specifically** (KML, QML): extract the
   relevant function(s) via regex + brace-matching into a standalone Node
   script (mocking any browser/Turf calls it depends on), call it with
   representative fake data, and run a small XML tag-balance parser over the
   output. For anything involving CSV quoting/Excel-formula tricks, also
   round-trip the output through a hand-written RFC4180 parser to confirm
   what a real CSV reader (or Excel) would actually see — don't just eyeball
   the generated string.

---

## 15. General lessons for porting this to a new environment

- **One data-driven layer catalogue, many small generic functions reading
  it** — resist the urge to special-case individual layers by name; add a
  property to the layer-def object instead.
- **When a third-party file format's behavior doesn't match the docs (or
  there are no docs), read the actual reader source** for that format if
  it's open source. This resolved two separate, otherwise-unexplainable QGIS
  bugs (label field schema, obstacle default) that speculative fixes had
  already failed to resolve.
- **Mutual-exclusion bugs between two features are almost always a missing
  shared gate**, not a defect in either feature's own logic — look for the
  boolean that should exist but doesn't before rewriting either side.
- **If a symptom can't be conclusively root-caused by static reading**,
  prefer converging on an already-proven-good pattern elsewhere in the same
  codebase over inventing a new, harder-to-verify one.
- **Every export/output format needs its own explicit checklist of geometry
  sources** (real data layers, synthetic/derived geometry like an AOI
  boundary, anything hand-drawn) — it's easy for one format's pipeline to
  silently omit a source another format already handles.
- **A file format's limitations are worth surfacing plainly** (e.g. "CSV
  cannot store bold text") rather than either refusing the request outright
  or silently faking an approximation that changes the file's actual
  content in a way the person didn't ask for.
- **Verify generated output programmatically**, not by inspection alone —
  especially for binary/structured formats (Shapefile, GeoPackage) and
  strict XML (KML, QML) where a single malformed attribute silently breaks
  the whole file in the target application, often with no error message at
  all.
