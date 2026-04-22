---
name: network-kmz-analysis
description: Analyze fiber route separation from two uploaded KMZ files. Use this skill whenever the user uploads KMZ or KML files and wants to check fiber route separation, parallel path spacing, minimum separation compliance, route conflict detection, or violation reporting between two fiber routes, duct banks, or network paths. Trigger when the user mentions KMZ, KML, fiber separation, route spacing, parallel fiber analysis, or asks if two routes are too close together.
---

# Network KMZ Fiber Separation Analysis Skill

Analyze two uploaded KMZ files (one per fiber route) and produce a detailed violation report identifying segments where the routes fall below a user-defined minimum separation threshold.

---

## Workflow

### Step 1 — Collect Inputs

Ask the user for the following if not already provided:

1. **KMZ File A** — Upload the first fiber route KMZ file
2. **KMZ File B** — Upload the second fiber route KMZ file
3. **Minimum separation threshold** — e.g., `50 feet`, `25 meters`
4. **Unit preference** — Feet or Meters (for the output report)

If files are already uploaded, confirm which file corresponds to which route before proceeding.

---

### Step 2 — Parse the KMZ Files

KMZ files are ZIP archives containing a `.kml` file (and optionally images/overlays). Use Python to extract and parse them. **Always read the KML content as bytes and pass to `ET.fromstring()` — do not use `ET.parse()` directly on the file handle inside a ZipFile, as this can fail on some KMZ structures.**

```python
import zipfile, math
from xml.etree import ElementTree as ET

def parse_kmz(kmz_path):
    """Extract LineString coordinates from a KMZ file."""
    with zipfile.ZipFile(kmz_path, 'r') as z:
        kml_name = [f for f in z.namelist() if f.endswith('.kml')][0]
        with z.open(kml_name) as kml_file:
            content = kml_file.read()  # Read as bytes first
    root = ET.fromstring(content)

    segments = []
    for placemark in root.iter('{http://www.opengis.net/kml/2.2}Placemark'):
        name_el = placemark.find('.//{http://www.opengis.net/kml/2.2}name')
        name = name_el.text if name_el is not None else 'Unnamed'
        for coords_el in placemark.iter('{http://www.opengis.net/kml/2.2}coordinates'):
            coords_text = coords_el.text.strip()
            points = []
            for token in coords_text.split():
                parts = token.split(',')
                if len(parts) >= 2:
                    try:
                        lon, lat = float(parts[0]), float(parts[1])
                        points.append((lat, lon))
                    except ValueError:
                        pass  # Skip malformed tokens
            if len(points) >= 2:
                segments.append({'name': name, 'points': points})
    return segments
```

**Known real-world patterns (validated against Columbia Route KMZ files):**
- Route A may be a single long LineString (45 points, ~10 miles)
- Route B may be many short overlapping segments (26 segments, 2–232 points each) — this is normal for routes drawn segment-by-segment in Google Earth
- Segment names may all be `'0'` or generic — don't rely on names for identification; use filename as route label
- Always use the **filename** (not internal segment names) as the route label in the report

**Edge cases to handle:**
- KML files with no namespace prefix — use `iter()` with full tag URI `{http://www.opengis.net/kml/2.2}`
- MultiGeometry placemarks — flatten into individual LineStrings
- Point-only placemarks — skip (only analyze LineStrings with 2+ points)
- Missing or malformed coordinate tokens — skip with `try/except ValueError`

---

### Step 3 — Compute Separation via Densified Sampling

Densify Route A to ~5m intervals before analysis. This ensures accurate violation detection even when original KMZ points are sparse. Check each densified point against all Route B segments using point-to-segment distance.

```python
def haversine(lat1, lon1, lat2, lon2):
    """Return distance in meters between two lat/lon points."""
    R = 6371000
    phi1, phi2 = math.radians(lat1), math.radians(lat2)
    dphi = math.radians(lat2 - lat1)
    dlam = math.radians(lon2 - lon1)
    a = math.sin(dphi/2)**2 + math.cos(phi1)*math.cos(phi2)*math.sin(dlam/2)**2
    return 2 * R * math.asin(math.sqrt(a))

def point_to_segment_distance(p, a, b):
    """Minimum distance from point p to segment a→b (lat/lon tuples)."""
    ax, ay = a[1], a[0]
    bx, by = b[1], b[0]
    px, py = p[1], p[0]
    dx, dy = bx - ax, by - ay
    seg_len_sq = dx*dx + dy*dy
    if seg_len_sq == 0:
        return haversine(p[0], p[1], a[0], a[1])
    t = max(0, min(1, ((px - ax)*dx + (py - ay)*dy) / seg_len_sq))
    nearest_lat = a[0] + t*(b[0]-a[0])
    nearest_lon = a[1] + t*(b[1]-a[1])
    return haversine(p[0], p[1], nearest_lat, nearest_lon)

def interpolate_segment(points, step_m=5):
    """Resample a LineString to ~step_m intervals for accurate analysis."""
    result = [points[0]]
    for i in range(len(points)-1):
        a, b = points[i], points[i+1]
        seg_len = haversine(a[0], a[1], b[0], b[1])
        if seg_len == 0:
            continue
        n = max(1, int(seg_len / step_m))
        for k in range(1, n+1):
            t = k / n
            result.append((a[0] + t*(b[0]-a[0]), a[1] + t*(b[1]-a[1])))
    return result

def nearest_distance_to_route(point, route_segments):
    min_d = float('inf')
    for seg in route_segments:
        pts = seg['points']
        for j in range(len(pts)-1):
            d = point_to_segment_distance(point, pts[j], pts[j+1])
            if d < min_d:
                min_d = d
    return min_d
```

---

### Step 4 — Identify and Group Violations

A **violation** is any sampled point where separation < threshold. Group contiguous violation points into **violation zones**.

```python
METERS_PER_FOOT = 0.3048

def find_violation_zones(route_a_segments, route_b_segments, threshold_m):
    # Densify Route A
    all_points = []
    for seg in route_a_segments:
        for p in interpolate_segment(seg['points'], step_m=5):
            all_points.append(p)

    # Score each point
    results = []
    for p in all_points:
        d = nearest_distance_to_route(p, route_b_segments)
        results.append({'lat': p[0], 'lon': p[1], 'dist_m': d, 'violation': d < threshold_m})

    # Group into zones
    violations, in_zone, zone = [], False, None
    for r in results:
        if r['violation']:
            if not in_zone:
                zone = {'start_lat': r['lat'], 'start_lon': r['lon'],
                        'min_dist_m': r['dist_m'], 'points': [r]}
                in_zone = True
            else:
                zone['points'].append(r)
                if r['dist_m'] < zone['min_dist_m']:
                    zone['min_dist_m'] = r['dist_m']
        else:
            if in_zone:
                _close_zone(zone, violations, threshold_m)
                in_zone, zone = False, None
    if in_zone and zone:
        _close_zone(zone, violations, threshold_m)
    return violations, results

def _close_zone(zone, violations, threshold_m):
    pts = zone['points']
    zone['end_lat'] = pts[-1]['lat']
    zone['end_lon'] = pts[-1]['lon']
    zone['length_m'] = sum(haversine(pts[i]['lat'], pts[i]['lon'],
                                      pts[i+1]['lat'], pts[i+1]['lon'])
                           for i in range(len(pts)-1))
    zone['length_ft'] = zone['length_m'] / METERS_PER_FOOT
    zone['min_dist_ft'] = zone['min_dist_m'] / METERS_PER_FOOT
    zone['buffer_needed_ft'] = (threshold_m / METERS_PER_FOOT) - zone['min_dist_ft']
    zone['is_terminal'] = zone['length_m'] < 10  # Flag single-point / endpoint violations
    violations.append(zone)
```

---

### Step 5 — Aggregate Violations into Report

Violation zones are already grouped in Step 4. For each zone, report all five fields. Additionally, flag **terminal endpoint violations** — these occur when the violation zone length is < 10m, indicating the two routes converge only at a shared facility endpoint (POP, carrier hotel, data center). This is operationally expected and should be called out distinctly rather than treated the same as a mid-route separation failure.

---

### Step 6 — Output the Violation Report

Present results in this order:

1. **Summary header** — Route A filename, Route B filename, threshold, unit, sampled points analyzed, violation zones found, min/max separation observed, total route length in violation
2. **Violation table** — One row per violation zone with all fields from Step 4
3. **Assessment** — Interpret the results:
   - If `is_terminal=True`: note this is likely a shared facility convergence, which is operationally expected
   - If mid-route violations exist: call out severity and recommend rerouting
4. **Clean bill of health** — If zero violations, confirm compliance explicitly

**Format:** Present inline in chat. If more than 20 violation zones, also generate a CSV and offer to export it.

**Report field reference:**

| Field | Description |
|---|---|
| **Routes in Conflict** | Route A filename vs Route B filename |
| **Violation Start Coords** | Lat/Lon of first point in the violation zone |
| **Violation End Coords** | Lat/Lon of last point in the violation zone |
| **Closest Approach** | Minimum distance in this zone (ft and m) |
| **Zone Length** | Cumulative length along Route A (ft and m) |
| **Buffer Needed** | Additional separation required to reach threshold |
| **Terminal Flag** | ⚠️ "Endpoint convergence" if zone length < 10m |

---

## Unit Handling

- Internally always compute in **meters**
- Convert for display based on user's unit preference:
  - Feet: `meters × 3.28084`
  - Meters: as-is
- Threshold entered by user: parse the unit from their input (e.g., `50 feet`, `15m`, `25 meters`)

---

## Error Handling

| Situation | Response |
|---|---|
| KMZ contains no LineStrings | Warn user; ask if they meant to upload a different file |
| KMZ is not a valid ZIP/KML | Inform user the file appears corrupt or is not a KMZ |
| Routes don't overlap geographically | Report "No proximity detected — routes may be in different geographic areas" |
| Only one file uploaded | Prompt for the second file before proceeding |
| Threshold not specified | Ask before running analysis |

---

## Example Invocation

> "Here are two KMZ files for our primary and diverse fiber routes in the Newark–NYC corridor. Check if they maintain at least 150 feet of separation anywhere along the parallel run."

**Validated real-world result (Columbia Route 1 vs Route 2, NJ/NYC corridor):**
- Route 1: 1 segment, 45 points, ~10.4 miles
- Route 2: 26 segments (typical for Google Earth drawn routes), 2-232 points each
- 3,330 densified sample points analyzed
- 1 violation zone detected: terminal endpoint convergence at 40.842730, -73.941957
- Closest approach: 136.5 ft (13.5 ft below 150 ft threshold)
- Zone length: ~0 ft (single-point terminal event — routes converging at shared POP)
- Assessment: operationally expected convergence at shared facility; not a diversity failure

Expected flow:
1. Parse both KMZ files (read bytes then ET.fromstring())
2. Densify Route A to 5m intervals (~3,000+ sample points for a 10-mile route)
3. Check each sample point against all Route B segments
4. Group violations into zones; flag terminal convergences
5. Return violation report with summary, zone table, and operational assessment
