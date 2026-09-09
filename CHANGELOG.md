# Changelog

## v1.0.0 - 2026-09-09

Initial release.

- `de-plz.geojson`: 8,173 German 5-digit PLZ polygons, EPSG:4326, 4.77 MB.
- Properties reduced to `plz` (5-digit string) and `name` (code plus place name).
- Built from `tdudek/de-plz-geojson` (OSM/ODbL) with mapshaper 0.7.59:
  `-clean`, `-simplify visvalingam 55% keep-shapes`, `-o precision=0.00001`.
- Simplification level selected by measuring area distortion across algorithms rather than by
  file size alone. Median area error 0.84%, p95 5.78%.

### QA added after the initial build (same file, no geometry change)

- Coverage completeness verified against the GeoNames DE postal code list: no geographic holes.
  2,651 of the 2,652 absent codes are Großempfänger codes with no delivery area; the remaining one
  is `87491 Jungholz`, in Austria.
- 12 codes could not be confirmed against a current authoritative list. Bounded at 0.19% of mapped
  area; cannot cause a wrong join.
- Topology verified: 97.5% of arcs are shared between neighbours, net area change -0.007%, and
  grow/shrink is balanced, so simplification opened no gaps or overlaps.
