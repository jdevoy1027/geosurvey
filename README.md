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

## Not built yet

The whole point of the application: recording observations. When you are ready,
the pieces are roughly

- a point capture button that stamps the current fix with its accuracy,
- a form for the attributes being collected, which needs to know what they are,
- photos with the point, which on iOS means `<input type="file" accept="image/*" capture="environment">`,
- local persistence (IndexedDB) so a day in the field survives a reload and a
  dead zone, and
- export — GeoJSON out, and whatever route gets it into the desktop GIS.

Worth deciding early: whether this needs to work with **no signal at all**. That
is the difference between a web page and an offline-capable one, and it changes
the base map story completely — NAIP is rendered on demand and cannot work
offline, so offline means pre-caching tiles for a known survey area before
leaving.
