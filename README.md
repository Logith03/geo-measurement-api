# Geospatial File Measurement API

A FastAPI service that accepts a **KML** file or a **zipped Shapefile**, extracts every feature
(index, geometry type, geometry, CRS, attributes) and computes **area** (polygons) and **length**
(lines) in square metres / metres. Measurements are never computed in degrees: geometries are
projected to a suitable metric CRS first.

## Setup

Requires Python 3.10+.

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
pytest                                           # run the tests
uvicorn app.main:create_app --factory --reload   # start the API on :8000
```

Interactive docs: <http://localhost:8000/docs>. Try it with the bundled sample:

```bash
curl -F "file=@samples/sample.kml" http://localhost:8000/api/files/
```

Configuration (environment variables, all optional):

| Variable | Default | Meaning |
|---|---|---|
| `GEO_API_DATA_DIR` | `data` | Where the SQLite DB and raw uploads are stored |
| `GEO_API_MAX_UPLOAD_MB` | `50` | Max upload size |
| `GEO_API_MAX_UNCOMPRESSED_MB` | `500` | Max total size of a Shapefile archive once unzipped |

## API

| Method & path | Purpose |
|---|---|
| `POST /api/files/` | Upload (`multipart/form-data`, field `file`) and process a `.kml` or `.zip` |
| `GET /api/files/{id}/` | File info and status |
| `GET /api/files/{id}/measurements/` | Per-feature measurements + totals (`limit`, `offset`) |
| `GET /api/files/{id}/features/` | Features with geometry, CRS, properties and measurements (`limit`, `offset`, `include_geometry`) |
| `GET /health` | Liveness check |

**Upload** → `201 Created`

```json
{
  "id": "9f2c5e0b7a3d4c1e8b6a2f4d1c0e9b87",
  "filename": "sample.kml",
  "file_type": "KML",
  "status": "COMPLETED",
  "feature_count": 4,
  "crs": "EPSG:4326",
  "error": null,
  "size_bytes": 1650,
  "created_at": "2026-10-08T10:00:00+00:00"
}
```

**Measurements** (numbers below are illustrative; real values depend on the file)

```json
{
  "file_id": "9f2c5e0b7a3d4c1e8b6a2f4d1c0e9b87",
  "crs": "EPSG:4326",
  "limit": 1000, "offset": 0,
  "summary": {
    "feature_count": 4,
    "counts_by_status": {"MEASURED": 2, "NOT_REQUIRED": 1, "UNSUPPORTED": 1},
    "total_area_m2": 1100000.0,
    "total_length_m": 1500.0
  },
  "measurements": [
    {"feature_index": 0, "source_id": "plot-1", "geometry_type": "Polygon", "status": "MEASURED",
     "area_m2": 1100000.0, "length_m": null, "measurement_crs": "LAEA(lat_0=13.0850, lon_0=80.2750)", "notes": []},
    {"feature_index": 1, "geometry_type": "LineString", "status": "MEASURED",
     "area_m2": null, "length_m": 1500.0, "measurement_crs": "EPSG:32644", "notes": []},
    {"feature_index": 2, "geometry_type": "Point", "status": "NOT_REQUIRED", "notes": []},
    {"feature_index": 3, "geometry_type": "GeometryCollection", "status": "UNSUPPORTED",
     "notes": ["Measurement is not supported for geometry type GeometryCollection"]}
  ]
}
```

Per-feature `status`: `MEASURED`, `NOT_REQUIRED` (points), `UNSUPPORTED` (no/empty geometry or an
unsupported type such as a mixed collection), `ERROR` (measurable type but the computation failed).
Problems are reported in `notes`; one bad feature never fails the file.

**Errors**: `400` empty file · `404` unknown id · `409` results requested for a non-`COMPLETED` file ·
`413` too large · `415` unsupported extension · `422` unreadable file (the file is still recorded with
`status: FAILED` and an `error`; the `file_id` is in the response detail).

## Architecture

```
app/
  main.py              app factory, wiring
  config.py            settings from env
  db.py                SQLite repository (files, features)
  schemas.py           response models
  api/files.py         HTTP layer only (validation, status codes)
  services/
    readers.py         KML + Shapefile parsing -> neutral RawFeature objects
    crs.py             CRS naming, UTM/LAEA selection
    measurements.py    Measurer: projection + area/length
    processing.py      orchestration: read -> build geometry -> measure
```

**File-processing flow.** Validate extension/size → store the raw upload under a server-generated
name → create a `files` row (`PROCESSING`) → parse (`readers`) → build shapely geometries and measure
each feature → store all features and flip the file to `COMPLETED` in one transaction. A file-level
failure (corrupt zip/XML, missing `.prj`, no features) marks it `FAILED`. Feature-level problems
become per-feature notes.

**Measurement flow.** Source CRS → WGS84 lon/lat (used only to *locate* the feature, via the
bounding-box centre) → a projected CRS chosen for that feature → planar `area` / `length` in metres.

**CRS handling.**
- KML is EPSG:4326 by specification. A Shapefile's CRS is read from its `.prj`; a missing or
  unparseable `.prj` is rejected rather than guessed. Projected source CRSs work too.
- **Areas** use a Lambert Azimuthal Equal-Area projection centred on the feature. Equal-area means
  area is preserved by construction, with no UTM-zone edge effects.
- **Lengths** use the UTM zone of the feature (UPS near the poles). UTM is conformal and accurate
  to roughly 0.04 % within a zone, which is fine for lengths.
- Axis order is forced to lon/lat (`always_xy=True`) everywhere to avoid the classic lat/lon swap.
- The `measurement_crs` of each feature is returned so results are auditable.

## Design decisions

- **FastAPI over Django.** The task is a small, stateless API; FastAPI gives validation, OpenAPI docs
  and a tiny footprint. Django would pay off if we needed auth, admin or a PostGIS-backed ORM.
- **Parsing KML myself (lxml) and Shapefiles with pyshp**, instead of GDAL/Fiona/GeoPandas. KML is
  simple XML and the pure-Python/wheel-only dependency set installs everywhere with no system GDAL.
  The trade-off is narrower format coverage (no KMZ, GeoPackage, etc.); the reader layer is isolated
  so GDAL/pyogrio could replace it later.
- **Per-feature projected CRS** rather than one CRS per file: more accurate for files that span
  several UTM zones, at a small CPU cost.
- **Projected measurement vs. pure geodesic** (`pyproj.Geod`). Geodesic is arguably the most accurate,
  but the brief asks for projection-based measurement, and the tests use geodesic results as the
  reference to verify the projected numbers (within 0.1 %).
- **Synchronous processing.** Simple and returns the result immediately; fine for typical survey files.
  The `status` field and the file/feature split already model an async flow.
- **SQLite + stdlib `sqlite3`.** Zero setup, transactional, enough for a single node.
- **Safety.** KML parsed with entity expansion/DTD/network disabled; Shapefile zips are read in memory
  (no path traversal) with a hard cap on uncompressed size; uploads are stored under generated names.
- **Altitude is ignored**; KML Z values are dropped. Only 2-D measurements are produced.

## Known limitations

- Features that cross the antimeridian or enclose a pole are not handled specially.
- Self-intersecting polygons are measured as-is and flagged in `notes`, not repaired.
- One Shapefile per zip; KMZ and `gx:Track`/3-D models are not supported.
- Auth, rate limiting and request-level timeouts are out of scope.

## Learnings and future scope

**Learnings**
- Degrees are not metres: choosing the right projection per purpose (equal-area for area, conformal
  for length) matters more than the library used.
- Validating against an independent reference (geodesic) is the best way to trust CRS code.
- Treating features as individually fallible keeps one malformed Placemark from rejecting a file.

**Future scope**
- Background processing (Celery/RQ) with polling or webhooks for large files.
- KMZ, GeoJSON and GeoPackage support; GDAL/pyogrio reader as an alternative backend.
- PostGIS storage with spatial queries; geometry simplification for large responses.
- Optional geodesic measurements and perimeter output; antimeridian handling; geometry repair.
- Auth, per-user files, delete endpoint, Dockerfile and CI, streaming/pagination of huge files.

## Testing

`pytest` covers the measurement maths (against geodesic references, including a projected source
CRS), CRS selection, KML and Shapefile parsing edge cases, and the API end to end.
