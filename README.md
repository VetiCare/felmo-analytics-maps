# de-plz-geojson

German 5-digit postal code (PLZ) polygons as a single GeoJSON file, simplified for use as a
Metabase custom region map.

## The files

`de-plz.geojson` is all of Germany: 8,173 features, 4.77 MB, EPSG:4326.

The rest are regional subsets, each a radius around a centre point. Every one is a byte-identical
subset of the Germany file and carries the same two properties.

| file | centre (lat, lon) | radius | PLZ | size |
|---|---|---|---|---|
| `de-plz-berlin.geojson` | 52.4986, 13.3662 | 45 km | 261 | 112 KB |
| `de-plz-bielefeld.geojson` | 52.0146, 8.5631 | 35 km | 84 | 48 KB |
| `de-plz-bremen.geojson` | 53.0826, 8.7984 | 40 km | 87 | 51 KB |
| `de-plz-erfurt.geojson` | 50.9841, 11.0346 | 40 km | 73 | 57 KB |
| `de-plz-halle.geojson` | 51.3567, 12.3294 | 60 km | 190 | 137 KB |
| `de-plz-hamburg.geojson` | 53.5879, 10.0053 | 50 km | 295 | 140 KB |
| `de-plz-hannover.geojson` | 52.3732, 9.7516 | 45 km | 148 | 85 KB |
| `de-plz-koeln.geojson` | 50.9223, 7.0130 | 45 km | 263 | 116 KB |
| `de-plz-magdeburg.geojson` | 52.1190, 11.6223 | 30 km | 37 | 29 KB |
| `de-plz-mannheim-heidelberg.geojson` | 49.4500, 8.5198 | 50 km | 335 | 156 KB |
| `de-plz-muenchen.geojson` | 48.1458, 11.5750 | 45 km | 278 | 134 KB |
| `de-plz-nuernberg.geojson` | 49.4465, 11.0894 | 50 km | 236 | 138 KB |
| `de-plz-rhein-main.geojson` | 50.0860, 8.5707 | 55 km | 388 | 183 KB |
| `de-plz-rhein-ruhr.geojson` | 51.4287, 6.9540 | 65 km | 529 | 230 KB |
| `de-plz-rostock.geojson` | 54.1094, 12.0587 | 40 km | 38 | 35 KB |
| `de-plz-stuttgart.geojson` | 48.7656, 9.1750 | 35 km | 242 | 104 KB |

### Properties

Every feature in every file has exactly two:

| property | example | purpose |
|---|---|---|
| `plz`  | `"01067"` | 5-digit code as a **string**, leading zeros preserved. Join key. |
| `name` | `"01067 Dresden"` | Code plus place name, for the tooltip. |

### Why regional files exist

Metabase custom region maps cannot zoom. `LeafletChoropleth.tsx` hard-disables `dragging`,
`zoomControl`, `scrollWheelZoom`, `doubleClickZoom`, `boxZoom`, `touchZoom` and `keyboard`, then
calls `fitBounds` once using bounds computed from **every feature in the GeoJSON file**. Still true
on master as of v0.63.16. A smaller file is therefore the only way to get a city-scale view, and a
separate card per file the only way to switch between them.

Each regional file is a plain radius around a city centre, listed above. Widen or add one by
re-running the build against `de-plz.geojson`.

## Metabase setup

Admin > Settings > Maps > Custom Maps > Add a Map

| field | value |
|---|---|
| Name of the map | `Deutschland PLZ (5-stellig)` |
| URL | the raw/CDN URL of `de-plz.geojson` in this repo |
| Region's identifier | `plz` |
| Region's display name | `name` |

Your query must return `plz` as a 5-digit **string**. A numeric column drops leading zeros and the
622 codes starting with `0` will not join.

### URL requirements (Metabase v0.60)

Verified against `src/metabase/geojson/settings.clj` and `api.clj` at tag `v0.60.26`:

- `http://` or `https://` only, and the host must resolve to a public, externally routable address.
  Internal hosts, localhost and `file://` are rejected.
- Redirects are **not** followed (`:redirect-strategy :none`). Use `raw.githubusercontent.com`
  directly; a `github.com/<owner>/<repo>/raw/...` URL returns 302 and fails.
- Content-type must begin with `application/geo+json`, `application/vnd.geo+json`,
  `application/json` or `text/plain`. GitHub raw sends `text/plain`, jsDelivr sends
  `application/json`. Both are accepted.
- 8 second connect and socket timeout.
- v0.60.26 has no server-side cache for custom GeoJSON, so every map render refetches from the
  URL. Prefer a CDN URL and pin a tag so a bad push cannot break production maps.
- The 5 MB figure in the Metabase docs is a recommendation. There is no size check in the code.

## How the file was built

Source: [tdudek/de-plz-geojson](https://github.com/tdudek/de-plz-geojson) `plz-5stellig.geojson`
(OpenStreetMap derived, ODbL). Original source: [postleitzahl.net](https://www.postleitzahl.net/plz-downloads).

```bash
npm i -g mapshaper   # 0.7.59

curl -sSLO https://raw.githubusercontent.com/tdudek/de-plz-geojson/master/plz-5stellig.geojson

mapshaper plz-5stellig.geojson \
  -each 'name = note' \
  -filter-fields plz,name \
  -clean \
  -simplify visvalingam 55% keep-shapes \
  -o precision=0.00001 format=geojson de-plz.geojson
```

`visvalingam 55%` was chosen by sweeping algorithms and levels and measuring area distortion
against the unsimplified source. It gave the lowest median and p95 error of the options that fit
under 5 MB.

| variant | size | median area err | p95 | polys >25% err |
|---|---|---|---|---|
| Douglas-Peucker 55% | 4.78 MB | 1.13% | 6.80% | 7 |
| **Visvalingam 55%** | **4.77 MB** | **0.84%** | **5.78%** | **11** |
| default (weighted) 50% | 4.51 MB | 1.14% | 7.24% | 22 |
| default (weighted) 60% | 5.01 MB | 0.81% | 5.43% | 6 |

The upstream data is already generalized (328k vertices, about 40 per polygon), so file size is
driven mainly by coordinate precision. Rounding to 5 decimals (about 1 m) cut the file from
9.4 MB to 7.0 MB with no shape loss before any simplification.

## Validation

| check | result |
|---|---|
| features | 8,173 |
| `plz` is a 5-digit string | 8,173 of 8,173, 622 with a leading zero |
| duplicate `plz` | 0 |
| missing or non-string `name` | 0 |
| geometry types | 7,917 Polygon, 256 MultiPolygon |
| unclosed rings | 0 |
| self-intersections | 72 repaired by `-simplify`, 1 sliver removed by `-clean` |
| CRS | no `crs` member, so the GeoJSON default WGS84 (EPSG:4326) |
| bbox | 5.8667, 47.2702 to 15.0381, 55.0587 |
| max coordinate decimals | 5 |
| size | 4,998,492 bytes (4.77 MB), 1.08 MB gzipped |

Shape fidelity was checked by overlaying the simplified outlines on the source at city zoom for
München (74 polygons) and Berlin (240 polygons). The 11 polygons with more than 25% area error are
all small downtown codes between 0.43 and 3.23 km², against a 27.2 km² median.

### Completeness of the polygon set

The polygon set was compared against the GeoNames German postal code list (10,813 codes, an
independent non-OSM lineage).

- 2,652 GeoNames codes have no polygon here. A point-in-polygon test places 2,651 of them inside
  an existing polygon, so they are Großempfänger codes (a single large recipient such as
  `01053 Commerzbank AG`) with no delivery area of their own. They are correctly absent.
- The one exception is `87491 Jungholz`, an Austrian village that uses a German postal code. It is
  outside Germany, so a German boundary dataset has no polygon for it.
- **The polygon set has no geographic gaps.**
- 12 codes here are absent from GeoNames and could not be confirmed against a current
  authoritative list, since Deutsche Post does not publish one freely: `04861`, `06485`, `06711`,
  `06772`, `09434`, `22961`, `25867`, `33333`, `39628`, `64760`, `99090`, `99095`. They are either
  retired since the 2019 upstream snapshot or GeoNames gaps. Bounded impact: 689 km², 0.19% of the
  mapped area. Because there are no holes, the worst case is that such a polygon never receives a
  row and renders blank. It cannot cause a wrong join.

### Topology

Simplification was checked for gaps and overlaps along shared borders.

| check | result |
|---|---|
| re-running `-clean` on the output | 8,173 of 8,173 retained, nothing further to fix |
| arcs shared by 2 or more polygons | 24,620 of 25,249 (97.5%). Internal borders are stored once, so both neighbours index the identical vertex list and a gap is not representable |
| arcs used by one polygon | 629 (2.5%), the national border, coastline and enclaves |
| net total area change | -0.007% (355,130 to 355,106 km²) |
| grew vs shrank | 3,934 grew, 4,239 shrank. Balanced, so shared borders moved for both neighbours rather than every polygon shrinking, which is what would open gaps |

## Scope

This repo holds OpenStreetMap derived geometry only. Do not add anything else: no metrics, no
lists, no per-area attributes. A Metabase custom map URL is readable by anyone who can reach the
instance, so treat everything here as public. Join keys and metrics belong in the warehouse.

## Licence

Contains information from OpenStreetMap, made available under the
[Open Database License](https://opendatacommons.org/licenses/odbl/) (ODbL).
© OpenStreetMap contributors. Derived work, so it stays ODbL. Keep this attribution.
