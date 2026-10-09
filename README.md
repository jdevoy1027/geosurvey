# Geosurvey

A field-data GIS for the phone. Same mapping stack as taxliencode.pro — Mapbox
GL JS 3.9.0, no build step, one self-contained `index.html` — but laid out for a
phone held in one hand rather than a desktop with a layer panel.

This first cut is the base to build on: the three base maps, and a GPS fix
reported precisely enough to judge whether a reading is worth recording. Data
capture is not built yet.

## What is here

| | |
|---|---|
| `index.html` | the whole application |

### Base maps

All three from the desktop GIS, switched by toggling raster layers rather than
by `setStyle` — `setStyle` tears the style down and rebuilds it, which would
drop every source and layer added since, including, once capture exists, the
points being collected.

- **Light** — `mapbox://styles/mapbox/light-v11`. The vector base; the only one
  that works outside the United States.
- **Satellite** — `mapbox://mapbox.satellite`, worldwide, mixed capture dates.
- **Aerial** — USDA NAIP / USGS orthoimagery at about 0.3 m, from zoom 12 up,
  with Satellite drawn beneath it so there is never a blank frame while a tile
  renders. Measured round-trip for one tile: **10.7 s**. It is rendered on
  demand by a USGS server, and the cost grows with the area requested, which is
  why it does not draw below z12 at all. Expect it to be slow on cellular.

### Location

Not Mapbox's `GeolocateControl`. The field question is "is this fix good enough
to record?", and a blue dot alone cannot answer it, so this reports latitude,
longitude, accuracy in metres and altitude, with the accuracy figure coloured
green under 10 m and red over 30 m. The accuracy disc is a real circle on the
ground: its pixel radius is recomputed from metres-per-pixel on every zoom,
otherwise it would claim a different accuracy at every zoom level.

`enableHighAccuracy: true` and `maximumAge: 0` — the GPS radio is worth the
battery here, and a cached fix from ten minutes ago is worse than none because
it looks exactly like a current one.

The button has three states: off, watching, and following. Panning by hand drops
out of following, so a deliberate gesture is not undone by the next fix.

### Phone specifics

`100dvh` rather than `100vh` (on iOS `100vh` is the height with the toolbars
hidden, so the bottom controls would sit under Safari's chrome); `viewport-fit=cover`
with `env(safe-area-inset-*)` so the map runs under the notch while the controls
stay clear of it; 44 px minimum touch targets; rotation and pitch disabled, since
a stray two-finger twist is easy to do and hard to undo, and a turned map makes a
recorded position harder to check against the ground; `apple-mobile-web-app-capable`
so Add to Home Screen opens it without browser chrome.

Everything above the bottom bar is positioned from the bar's **measured** height,
published to CSS by a `ResizeObserver`. The bar changes height when the base-map
note changes and when the phone is rotated; a fixed offset buried the locate
button and the scale bar behind it.

## Running it

### The Mapbox token will not work from localhost

This is the first thing that will stop you. The token is URL-restricted, and the
restriction is **endpoint-sensitive** — styles and fonts pass from anywhere, but
`/v4/` tile requests do not. Measured:

```
style   light-v11          localhost → 200      taxliencode.pro → 200
tile    mapbox.satellite   localhost → 403      taxliencode.pro → 200
```

So a localhost page draws the Light base map and then fails silently on
Satellite and on anything else tiled. Before developing locally, add the dev
origin at **account.mapbox.com → Tokens → the public token → URL restrictions**:

```
http://localhost:8080
```

Add whatever port you serve on — the restriction matches scheme, host *and*
port. Then add the real origin when you deploy.

The page detects this case: a 401/403 on the style replaces the map with a note
naming the origin that was refused.

### Serving

Any static server works; there is nothing to build.

```sh
cd /Users/joe/Documents/dev/Geosurvey
npx serve -l 8080 .
```

`python3 -m http.server 8080` also works, but note the system `python3` on this
machine is broken (the Command Line Tools are gutted — `xcode-select --install`
fixes it). `/usr/local/bin/python3` works today.

### Testing on the phone

Geolocation needs a **secure context**: `https://`, or `http://localhost` on the
device itself. Over plain http from another machine on the LAN the call fails
with a permission error that reads as the user having denied it. Options:

- deploy and test over https (simplest), or
- `npx localtunnel --port 8080`, or Tailscale/ngrok, for an https URL — remember
  to add that origin to the token too.

## Point capture

Green **+** records a point at the current fix. Press and hold on the map to
place one somewhere you cannot stand. Tap a recorded point to edit or delete it.
**List** shows them newest first and flies to whichever you pick; **Export**
hands out a GeoJSON file.

### Changing the form

The form *is* this array, near the bottom of `index.html`:

```js
var FIELDS = [
  { key: 'station', label: 'Station', type: 'text', required: true, auto: nextStation },
  { key: 'type',    label: 'Observation', type: 'select',
    options: ['Outcrop', 'Soil', 'Water', 'Structure', 'Sample', 'Photo point', 'Other'] },
  { key: 'notes',   label: 'Notes', type: 'textarea' }
];
```

Add, rename or remove a field there and the form, the stored records and the
export all follow — nothing else needs editing. `type` is `text`, `select`,
`number` or `textarea`; `auto` is a function returning the value to prefill on a
new point. The fields above are a placeholder for whatever you actually record;
they are a guess, not a recommendation.

Position, accuracy and time are deliberately **not** in `FIELDS`. They are
recorded automatically and cannot be edited — a field record whose coordinates
can be typed over is not a measurement.

### Two things it is careful about

**The fix is snapshotted when you press +**, not read again when you save.
Filling in a form takes a minute, and the position drifts in that minute; the
point belongs where you stood when you pressed the button. Verified: with the
form open, moving the simulated position 0.1° away left the saved coordinates
unchanged.

**A placed point is not a measurement.** Press-and-hold points are stored with
`src: "placed"` and a null accuracy, drawn as a hollow ring rather than a solid
dot, and the form says "placed by hand — no measured accuracy". They should
never be mistaken later for somewhere you stood.

Accuracy is coloured in the form as well as the readout: green at 10 m or
better, red at 30 m or worse.

### What a record looks like

```json
{
  "type": "Feature",
  "properties": {
    "src": "gps", "at": "2026-10-09T03:57:16.703Z",
    "station": "S001", "type": "Outcrop", "notes": "Sandstone, cross-bedded",
    "acc": 4.7, "alt": 58.2
  },
  "geometry": { "type": "Point", "coordinates": [-118.212945, 34.004312] }
}
```

`at` is when it was recorded, `acc` the GPS accuracy in metres, `alt` the
altitude in metres, `src` either `gps` or `placed`. Coordinates are rounded to
six decimals — about 0.1 m, far finer than any phone fix.

### Storage

Everything is held in `localStorage` as one GeoJSON FeatureCollection, which is
also exactly what Export writes, so exporting is a serialise rather than a
conversion and there is no second representation to drift.

**It survives a reload and a force-quit. It does not survive clearing site
data**, and Safari can evict storage from a site you have not opened in a while.
**Export at the end of each day.** A quota failure is reported rather than
swallowed, but the honest protection is getting the data off the phone.

Export prefers the iOS share sheet when the browser supports sharing files —
Mail, AirDrop, iCloud directly — and falls back to a download otherwise, since a
download on iOS lands in Files and is then awkward to move anywhere useful.

## Not built yet

- **Photos.** `<input type="file" accept="image/*" capture="environment">` opens
  the camera, but images cannot go in `localStorage` — that means IndexedDB and
  a real storage budget.
- **Offline.** Worth deciding before this gets used in anger. NAIP is rendered
  on demand and cannot work without a connection, so offline means pre-caching
  tiles for a known survey area before leaving, plus a service worker for the
  page itself. It is the difference between a web page and an application.
- **Getting points into the desktop GIS.** Export produces GeoJSON, which the
  faults map already reads; nothing automates the hand-off yet.
