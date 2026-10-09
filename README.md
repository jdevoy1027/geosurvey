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

**Add Features** in the panel records a point at the current fix. Press and hold
on the map to place one somewhere you cannot stand. Tap a recorded point to edit
or delete it. **List** shows them newest first and flies to whichever you pick;
**Export** hands out the GeoJSON and the photos.

### Changing the form

The form *is* this array, near the bottom of `index.html`:

```js
var FIELDS = [
  { key: 'station',     label: 'Station', type: 'text', required: true, auto: nextStation },
  { key: 'type',        label: 'Observation', type: 'select', options: [...] },
  { key: 'technician',  label: 'Technician', type: 'text', auto: lastTech },
  { key: 'description', label: 'Description', type: 'textarea' },
  { key: 'x_coord',     label: 'X (longitude)', ro: true, fill: c => +c.lon.toFixed(6) },
  { key: 'y_coord',     label: 'Y (latitude)',  ro: true, fill: c => +c.lat.toFixed(6) },
  { key: 'date',        label: 'Date', ro: true, fill: c => c.date },
  { key: 'picture_id',  label: 'Picture ID', ro: true, live: true, fill: c => c.photos.slice() }
];
```

Add, rename or remove a field and the form, the stored records and the export
all follow — nothing else needs editing.

| | |
|---|---|
| `type` | `text`, `select`, `number`, `textarea` |
| `auto` | fn returning the value to prefill on a new point |
| `ro` | shown but not editable; its value comes from `fill()` |
| `fill` | `fn(ctx)` run on save; ctx is `{lon, lat, acc, alt, src, date, photos}` |
| `live` | a read-only field recomputed on *every* save, not just the first |

**`x_coord`, `y_coord`, `date` and `picture_id` are read-only on purpose.** They
are measurements and bookkeeping, not opinions — a field record whose
coordinates can be typed over is not a measurement. They still show in the form
and appear as columns in the export, which is what a table wants. They are
filled before you save, so what is about to be recorded is visible rather than
only discoverable afterwards.

Editing a point keeps its original position and date; only `picture_id` is
recomputed, since photos can be added or removed at any time.

**Technician carries forward** from the last point recorded. One person usually
records all day, and retyping a name at every station is how it ends up spelled
three different ways in one survey.

Underscores rather than the hyphens in *x-coord* / *y-coord*: a hyphen is legal
in JSON but breaks the moment this goes into a shapefile or most desktop GIS
field names, which is where survey data usually ends up.

Records written before these fields existed are migrated in place on load — `at`
→ `date`, `notes` → `description`, `photos` → `picture_id`, and `x_coord`/
`y_coord` filled from the geometry — so there is never a half-empty column.

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
    "src": "gps", "station": "S001", "type": "Soil",
    "technician": "J. Devoy", "description": "Coarse sand",
    "x_coord": -118.212945, "y_coord": 34.004312,
    "date": "2026-10-09T07:22:13.427Z",
    "picture_id": ["p1791522521909-122700"],
    "acc": 4.7, "alt": 58.2
  },
  "geometry": { "type": "Point", "coordinates": [-118.212945, 34.004312] }
}
```

`acc` is the GPS accuracy in metres, `alt` the altitude in metres, `src` either
`gps` or `placed`. Coordinates are rounded to six decimals — about 0.1 m, far
finer than any phone fix. `picture_id` holds internal ids in the stored record
and becomes a filename list on export.

### Photos

**+ Photo** on the form opens the rear camera (the library too — a shot taken
earlier is often the one being filed). Tap a thumbnail to view it full screen,
or delete it from there.

Each photo is resized to **1600 px on the long edge** at JPEG 0.82 before it is
stored. A phone photo is 3–4 MB and about 4000 px across; nothing here needs
that, and a day of them at full size would not fit. Measured: a 3000×2000 test
image went in at 192 KB and came out 1600×1067 at 24 KB, aspect preserved.

Two consequences of the re-encode worth knowing. It applies the EXIF rotation
first — without that, a photo shot in portrait would be stored on its side,
because a canvas re-encode drops EXIF. And it **strips the camera's GPS tag**:
the point already carries a position that was recorded deliberately, and a
second, silent one buried in file metadata is a privacy leak waiting to be
shared.

### Identifying a tree with Pl@ntNet

With at least one photo on the point, pick the organ — **Auto, Leaf, Bark,
Flower, Fruit** — and tap **Identify with Pl@ntNet**. Up to five of that point's
photos go in one request (Pl@ntNet scores the *set*, so several angles of the
same tree genuinely help). Candidates come back with a confidence score; tap one
and it fills `species`, `common_name`, `id_score` and `id_source`.

The organ choice is remembered between points, since a survey tends to
photograph the same organ all day. **Bark** matters for tree work: on a mature
trunk out of reach of its own canopy, and in winter, it is often the only organ
to hand.

`species` and `common_name` stay **editable after an identification** — the
model proposes, the surveyor decides. `id_score` and `id_source` are recorded
alongside, because *Quercus agrifolia* at 0.91 from a bark photo and the same
name typed by hand are not the same claim, and six months later nothing else
would tell them apart. A score is the model's confidence, not proof.

**Your API key is never in this repository.** A Pl@ntNet key is tied to your
account and a daily quota, and this site is public — a key written into the page
would be readable by anyone who views source, and the first thing they could do
is spend your quota. Instead it is typed in once on the phone and kept in
`localStorage` on that device alone. Change it any time from **List → Pl@ntNet
key**.

The deliberate consequence: **each device needs the key once**. If this ever
goes to a crew who should not hold it, the fix is a proxy holding the key as a
server secret — the same Cloudflare Worker pattern already used to put a
password on geonomics.online — not embedding it here.

Pl@ntNet permits browser requests outright (it echoes the page's origin back in
`access-control-allow-origin`), so the app talks to the API directly and needs
no server of its own. Quota exhaustion returns 429 and resets at 00:00 UTC;
the app says so, and since the photos are on the record the point can be
identified later. The same is true out of signal.

### Storage

Points live in `localStorage` as one GeoJSON FeatureCollection. **Photos live in
IndexedDB**, referenced by id, because `localStorage` holds strings and gets
about 5 MB — one photo would eat it. Separate budgets mean a photo can never
push the notes out.

The app asks for persistent storage on load, so Safari is less likely to evict a
survey between outings. It may refuse; nothing depends on it being granted.

**List** shows what the survey is costing — photo count, megabytes used and the
quota — because on a phone storage is the binding constraint and it is invisible
until it runs out.

The FeatureCollection is exactly what Export writes, so exporting is a serialise
rather than a conversion and there is no second representation to drift.

**It survives a reload and a force-quit. It does not survive clearing site
data**, and Safari can evict storage from a site you have not opened in a while.
**Export at the end of each day.** A quota failure is reported rather than
swallowed, but the honest protection is getting the data off the phone.

### Export

One share: the GeoJSON **and every photo** as separate files. Photos are renamed
on the way out — `S001-1.jpg`, `S001-2.jpg`, beside the `S001` record, with
`picture_id` carrying those names as one `"; "`-joined string rather than an
array, because this is a table column and an array in a cell does not survive a
shapefile or a CSV. The internal ids stay inside the app, where a station can be
renamed at any time and a filename derived from it would go stale.

It prefers the iOS share sheet — Mail, AirDrop, iCloud directly — because a
download on iOS lands in Files and is then awkward to move anywhere useful.
`canShare` is asked with the real file list rather than a token one, since the
limit is on the whole payload: a survey with forty photos can be refused where
one file would pass. If it is refused, the GeoJSON downloads and you are asked
before the photos follow one at a time, rather than a dozen silent downloads
firing off.

## Not built yet

- **Offline.** Worth deciding before this gets used in anger. NAIP is rendered
  on demand and cannot work without a connection, so offline means pre-caching
  tiles for a known survey area before leaving, plus a service worker for the
  page itself. It is the difference between a web page and an application.
- **Getting points into the desktop GIS.** Export produces GeoJSON, which the
  faults map already reads; nothing automates the hand-off yet.
