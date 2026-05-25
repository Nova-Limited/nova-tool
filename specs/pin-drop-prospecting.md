# Pin Drop Prospecting Spec

**For a future nova-tool build session (est. 5-7 hrs).** Captures the contract, integration points, and edge cases for the "paste lat/lng → return building + tenant + CH match" flow. Save to `~/nova-tool/specs/pin-drop-prospecting.md`.

---

## 1. Decisions locked in

| Decision | Choice |
|---|---|
| Pin source | **Google Earth Pro** — user right-clicks the target building, copies coordinates to clipboard, pastes into nova-tool |
| Building footprint dataset | **Microsoft Global ML Building Footprints** (primary), **OSM Overpass** (fallback for gaps) — both ODbL, both free |
| Footprint delivery model | **Worker-mediated** — new `/pin-drop` endpoint on the existing nova-ch-proxy Worker; client never talks to Microsoft / OSM directly |
| Coordinate input formats accepted | **Decimal degrees** (`50.431375, -4.131317`) and **DMS** (`50°25'52.95"N, 4°7'52.74"W`). Auto-detect and normalise. |
| Roof area calculation | **Server-side in the Worker** via spherical excess formula on the WGS84 polygon. Single canonical value returned to the client. |
| GMB/Places lookup driver | **Reverse: lat/lng → nearest GMB place** (NOT name → places, which is what `/places` does today). Requires a small extension to the existing Worker `/places` route OR a new sub-route. See section 6. |
| CH lookup driver | **Returned GMB tenant name** fed into existing `/search/companies` flow with `_cleanCompanyName` applied first |
| MVP scope | **OSM Overpass first**, Microsoft as a v1.1 swap once the rest of the pipeline is proven |
| Partial-success policy | **Render whatever succeeded, mark the rest as "not found"** — user can still proceed manually if (e.g.) building outline came back but no tenant did |

---

## 2. Coordinate input — format, validation, normalisation

### 2.1 What Earth Pro emits

Google Earth Pro's right-click → Copy Coordinates produces a clipboard string controlled by Earth's "Show Lat/Long" setting under Tools → Options → 3D View. Two formats are common:

| Setting | Clipboard output example |
|---|---|
| Decimal degrees | `50.431375, -4.131317` |
| DMS (default install) | `50°25'52.95"N, 4°7'52.74"W` |
| Universal Transverse Mercator | `30 U 419812.5 5587945.3` |

We accept the first two. UTM we reject with a clear "switch Earth to decimal or DMS" message — converting UTM properly requires zone tables and isn't worth the bytes.

### 2.2 Input field

Single text input on the proposal card, labelled `Pin coordinates (from Google Earth)`. Placeholder: `50.431375, -4.131317 or 50°25'52.95"N, 4°7'52.74"W`. Sits above the existing "Search Company" button.

Adjacent button: `Drop pin`. Triggers the lookup chain.

### 2.3 Validation + normalisation

Client-side parser (no Worker round-trip for malformed input):

1. **Detect format** — if string contains `°`, treat as DMS. Else attempt decimal parse on the two comma-separated halves.
2. **DMS → decimal** — regex `(\d{1,3})°\s*(\d{1,2})['′]\s*(\d{1,2}(?:\.\d+)?)["″]([NS]),?\s*(\d{1,3})°\s*(\d{1,2})['′]\s*(\d{1,2}(?:\.\d+)?)["″]([EW])`. Apply sign from hemisphere letters. Earth Pro emits straight quotes (`'` `"`); spec the regex to tolerate curly (`′` `″`) too in case of clipboard reformatting.
3. **Range check** — UK only for now: `lat ∈ [49.5, 61.0]`, `lng ∈ [-8.5, 2.0]`. Outside this range → reject with "Coordinates outside UK — Nova currently only operates in the UK". Easy to widen later.
4. **Precision** — round to 6 decimal places (≈ 11cm). More precision is meaningless against the building polygon datasets.

### 2.4 Why client-side parse first

Bad coordinates are the most common failure mode. Catching them before the Worker round-trip saves ~2s of pointless network and gives the user instant feedback. The Worker still re-validates as defence-in-depth.

---

## 3. Microsoft Building Footprints — the reality check

### 3.1 What it actually is

**Not a REST API.** Microsoft's Global ML Building Footprints is a **dataset** released by Microsoft AI for Good. Distribution model:

- Hosted on Azure Blob Storage at `https://minedbuildings.z5.web.core.windows.net/global-buildings/`
- Organised by country, then by Bing Maps quadkey at zoom 9
- Each tile file is NDJSON-gzipped: one GeoJSON Feature per line, with Polygon geometry in WGS84
- Index file `UnitedKingdom.csv.gz` lists the quadkeys + URLs for the UK
- Licence: **ODbL 1.0** (attribution required, share-alike on derived datasets)
- No rate limits documented (Azure Blob Storage with public read; assume polite-use applies)

### 3.2 The integration friction

To answer "what building is at (50.431, -4.131)?" you have to:

1. Convert the lat/lng to a Bing zoom-9 quadkey (well-defined formula, ~20 lines of JS)
2. Fetch the per-quadkey NDJSON file (typically 5-50MB gzipped, 30-300MB raw)
3. gunzip-stream + line-by-line parse
4. Bounding-box pre-filter polygons against the query point
5. Precise point-in-polygon test on survivors
6. Return the polygon (or nearest within X metres if no exact match)

**On a Cloudflare Worker** this works but is **tight on memory** (128MB) and CPU (30s on paid plan). Mitigations:

- Stream the gzip rather than loading the whole file into memory
- Cache hot quadkeys in Cloudflare R2 (or Workers KV for small ones) with a 30-day TTL — building footprints change rarely
- Pre-warm common UK quadkeys (Plymouth area, prospect-rich regions) via a one-time cron job

### 3.3 Recommended MVP shortcut

Build the pipeline against **OpenStreetMap Overpass API** first, then swap in Microsoft Footprints once the rest of the chain is proven. Reasons:

- Overpass is a hosted REST API: `https://overpass-api.de/api/interpreter` with a single POST containing an Overpass QL query
- For "find building polygon containing point" the query is one line of Overpass QL
- Free, no auth, ~1 req/s sustained per IP (publicly run instances)
- ODbL licence same as Microsoft — no licensing change between MVP and v1.1
- UK building coverage in OSM is essentially complete for commercial / industrial / warehouse stock — exactly the prospecting target

Once the MVP is proven end-to-end, swap the Worker's footprint backend from Overpass to Microsoft. The Worker contract from the client's perspective stays identical (`{lat,lng}` in, polygon+area out).

### 3.4 Overpass query (MVP backend)

```
[out:json][timeout:10];
is_in({lat},{lng})->.a;
way(pivot.a)[building];
out geom;
```

`is_in` finds the area containing the point; `pivot` returns the smallest containing area; `[building]` filters to building tags; `out geom` returns the polygon as a sequence of `{lat, lon}` nodes.

Response shape (truncated):
```json
{
  "elements": [{
    "type": "way",
    "id": 12345678,
    "tags": { "building": "industrial", "name": "..." },
    "geometry": [
      { "lat": 50.4314, "lon": -4.1315 },
      ... (closing ring back to first node)
    ]
  }]
}
```

Convert the `geometry` array into a GeoJSON Polygon ring before passing to the area calculator.

---

## 4. Roof area from a polygon

### 4.1 Formula

Spherical excess on WGS84 (Earth radius 6,378,137 m):

```
area = | Σᵢ (λᵢ₊₁ − λᵢ) · (2 + sin(φᵢ) + sin(φᵢ₊₁)) | × R² / 2
```

Where λ is longitude in radians, φ is latitude in radians, and the sum is over consecutive vertex pairs around the closed ring.

Accurate to <0.1% for polygons up to a few km² at UK latitudes. Good enough for prospecting; over-engineering UTM-zoned planar area gains us nothing at this scale.

### 4.2 Multi-polygon edge cases

Microsoft footprints are typically single-Polygon Features. OSM occasionally returns MultiPolygon for buildings with detached wings or courtyards.

Handling:

| Geometry type | Area | Notes |
|---|---|---|
| Polygon, no holes | Outer ring | Standard case |
| Polygon, with holes | Outer ring − sum of inner rings | Courtyards, atria |
| MultiPolygon | Sum of all polygons (with holes deducted per polygon) | Wing + main building shared as one OSM way |

### 4.3 Sanity bounds

- Minimum: 50 m² (anything smaller is almost certainly a shed or fence segment, not a usable solar roof — reject as "too small, try a different pin")
- Maximum: 100,000 m² (anything bigger is almost certainly a multi-building cluster or industrial estate — flag as "polygon may include multiple buildings, verify visually" but still return)

### 4.4 Why server-side

Two reasons:
1. The polygon vertex list can be hundreds of points for an irregular industrial roof — saves payload to send only the computed area + a simplified outline back to the client
2. Keeps the calculator in one place where future tweaks (e.g. usable-roof fraction discount for skylights/HVAC) can land without touching the HTML

The Worker returns both: the polygon (for visual confirmation if we ever add a map preview) and the canonical area number.

---

## 5. GMB / Google Places tenant lookup

### 5.1 The reverse-lookup problem

The existing `/places` endpoint expects a **company name** and returns Place matches. For pin-drop, we have a **lat/lng** and need the Place AT that point.

Google Places API has the right primitive: **Nearby Search** with a small radius (e.g. 50m) around the dropped pin.

### 5.2 New Worker sub-route

Add `POST /places/nearby` to the Worker (sibling to the existing `POST /places`):

```
Request:  { lat: number, lng: number, radiusMeters?: number }
Response: { matches: [Place], confidence: "high" | "ambiguous" | "low" | "none" }
```

Confidence rules (mirror existing `/places` semantics):

| matches.length | confidence |
|---|---|
| 0 | "none" |
| 1 | "high" |
| 2-3 (top one >2× distance of next) | "high" |
| 2-3 (close distances) | "ambiguous" |
| 4+ | "low" — pin is on a multi-tenant building or shopping centre |

Match shape identical to existing `/places` output: `displayName`, `formattedAddress`, `nationalPhoneNumber`, `websiteUri`, `location: { latitude, longitude }`, `googleMapsUri`.

### 5.3 Default radius

50m. Reasoning: a pinned warehouse roof centroid will typically be within 50m of the Place pin Google records for the business at that address. Larger radii catch neighbouring buildings. Smaller miss the target on buildings with deep setbacks.

If `matches.length === 0` at 50m, retry once at 150m before giving up. Worker handles the retry internally; client only sees the final result.

### 5.4 Cost

Existing `/places` budget: 100 requests/day cap (matches Google Maps free tier). The new `/places/nearby` sub-route counts against the same cap. A pin-drop session that finds one tenant uses **1-2 Places calls** (potential retry at wider radius).

---

## 6. Companies House lookup from GMB tenant name

### 6.1 Input

The `displayName` of the top GMB match from section 5.

### 6.2 Cleaning

Apply existing `_cleanCompanyName(name)` helper (strips ` Limited`, ` Ltd`, ` PLC` suffixes case-insensitive). Reason: Google typically lists "Rittal-CSM" while Companies House has "RITTAL-CSM LIMITED" — without the suffix strip, the search matches by token similarity but the cleaned form is more reliable for exact-name dedupe downstream.

### 6.3 Search call

Existing `GET /search/companies?q={cleanedName}&items_per_page=10`. Reuses the response shape we already render in the Look up on CH panel.

### 6.4 Auto-pick rules

| CH result count | Action |
|---|---|
| 0 | Render the pin-drop result panel with CH section greyed out and message "No Companies House match — try Look up on CH manually" |
| 1 | Auto-attach. Calls `pickLookupResult` equivalent under the hood. Populates currentProspect with the CH number, type, status, registeredOffice. |
| 2+ | Show all results inside the pin-drop result panel using the existing dp-lookup-row markup. User picks. |

Auto-attaching the single-match case is the 30-second-prospect optimisation. The multi-match case still requires a click but at least skips the manual name typing.

---

## 7. UI flow in nova-tool

### 7.1 Where the input box lives

On the proposal card, between the dp-grid (Site Address + Sector) and the addr-buttons row. New section labelled `Pin drop` with the coordinate input and Drop pin button.

### 7.2 Sequence

1. **Idle** — user pastes coordinates. Drop pin button becomes enabled when input is non-empty.
2. **Validating** — client parses coordinates. Bad input → red text below input, button stays clickable for retry.
3. **Fetching footprint** — button → "Finding building…", spinner. Worker call to `/pin-drop` (the umbrella endpoint, see section 9).
4. **Fetching tenant** — same spinner, button → "Finding tenant…". Worker fans out to Places internally; client sees one combined response.
5. **Fetching CH** — button → "Matching Companies House…". Worker fans out to CH search.
6. **Render results panel** — fixed layout below the Drop pin button:

```
┌──────────────────────────────────────────┐
│ BUILDING                                 │
│ Roof area: 4,250 m²                      │
│ Coordinates: 50.4314, -4.1315            │
│                                          │
│ TENANT (Google Places)                   │
│ Rittal-CSM · 01752 723225                │
│ rittal-csm.co.uk                         │
│                                          │
│ COMPANIES HOUSE                          │
│ RITTAL-CSM LIMITED · No. 00858856        │
│ active · Inc. 1965                       │
│ [Use this match]                         │
└──────────────────────────────────────────┘
```

7. **Use this match** — clicked, populates currentProspect end-to-end: name, number, registeredOffice (via async /company/{num} enrich), address, phone, website, roofArea, location. Same downstream flow as a Search → Use this match today.

### 7.3 Partial-success rendering

Each section renders independently. If Places returns no matches, the TENANT section shows "No tenant found at this pin — try Search Company manually". The user can still take the roof area + coordinates forward; CH section is also skipped in that case.

If CH returns no match against the tenant name, the CH section shows "No Companies House match for 'Rittal-CSM' — try Look up on CH manually". User can still proceed with the tenant data.

If the building footprint itself fails (e.g. pin is on an empty field), the BUILDING section says "No building outline at this pin — verify coordinates" and the other sections still run on the pin location.

---

## 8. Error states

| Failure | Symptom | Message |
|---|---|---|
| Coordinates malformed | Client-side, before Worker call | "Coordinates not recognised. Paste decimal (`50.43, -4.13`) or DMS (`50°25'52"N, 4°7'52"W`)." |
| Coordinates outside UK | Client-side range check | "Coordinates outside UK — Nova currently operates in the UK only." |
| Building polygon: no match | Worker returns `building: null` | Section greyed: "No building outline at this pin." |
| Building polygon: too small (<50 m²) | Worker returns `building: { tooSmall: true }` | "Polygon found but only 32 m² — likely a shed or fence. Try a more central pin." |
| Places: no nearby tenant | Worker returns `places.matches: []` | Section greyed: "No Google Places tenant within 150m." |
| CH search: no match | Worker returns `ch.results: []` | Section greyed: "No Companies House match for 'X'." |
| CH search: 50+ matches | Worker returns `ch.results.length >= 50` | "Tenant name too generic — 50+ CH matches. Refine via Look up on CH manually." |
| Worker timeout (any layer) | 30s elapsed | "Pin drop timed out. Try again — Microsoft Building data is slow on cold cache." |
| Microsoft/Overpass upstream down | Worker returns 502 | "Building footprint service unavailable. The other layers may still work — re-pin in a moment." |
| Places daily cap (100) hit | Worker returns 429 on the places leg | TENANT section: "Daily Places cap reached. Try again tomorrow." Other sections still render. |

---

## 9. Worker contract — new `/pin-drop` endpoint

Single umbrella endpoint that fans out to footprint + places + CH internally and returns a combined result. Keeps the client simple (one fetch, one render) and lets the Worker parallelise where possible (Places + CH can fan out concurrently once tenant name is known; building footprint runs in parallel from the start).

### 9.1 Request

```
POST https://nova-ch-proxy.jamie-b41.workers.dev/pin-drop
Content-Type: application/json

{ "lat": 50.431375, "lng": -4.131317 }
```

### 9.2 Response (success — all three layers found)

```json
{
  "input": { "lat": 50.431375, "lng": -4.131317 },
  "building": {
    "found": true,
    "areaM2": 4250,
    "polygon": [[lng,lat], [lng,lat], ...],
    "source": "microsoft" | "osm-overpass",
    "sourceTags": { "building": "industrial" }
  },
  "places": {
    "confidence": "high",
    "matches": [ /* same shape as existing /places */ ]
  },
  "ch": {
    "results": [ /* same shape as existing /search/companies items */ ],
    "autoAttachIndex": 0
  },
  "timing": { "buildingMs": 1200, "placesMs": 800, "chMs": 400, "totalMs": 1450 }
}
```

`timing` is for telemetry / debugging the latency budget. Strip from production response if it ever leaks customer info — currently it doesn't.

`autoAttachIndex` is set to `0` when there's exactly one CH match, signalling the client to auto-attach. Set to `null` otherwise.

### 9.3 Response (partial failure)

```json
{
  "input": { "lat": 50.431375, "lng": -4.131317 },
  "building": { "found": true, "areaM2": 4250, ... },
  "places": { "confidence": "none", "matches": [] },
  "ch": { "results": [], "skipped": "no_tenant_name" },
  "timing": { ... }
}
```

CH leg is skipped when Places returns no matches — no name to search with. `skipped` field documents why.

### 9.4 Internal fan-out order

```
t=0   start /pin-drop
       ├── building fetch (Microsoft or Overpass)
       └── places /nearby fetch
                ↓
       (await places)
                ↓
t≈700ms   places returns → if matches.length > 0, start CH /search/companies on top match
                ↓
       (await both building + ch)
                ↓
t≈1200ms  combine, return
```

Building fetch is the long pole. Places + CH are sequential because CH depends on Places' output. Total budget: see section 10.

### 9.5 Caching

| Layer | Cache | TTL | Key |
|---|---|---|---|
| Microsoft quadkey file | R2 | 30 days | `{country}/{quadkey}.geojsonl` |
| Overpass response | Workers KV | 7 days | `overpass:{lat-rounded-4dp},{lng-rounded-4dp}` |
| Places `/nearby` result | Workers KV | 24 hours | `places-nearby:{lat-4dp},{lng-4dp},{radius}` |
| CH `/search/companies` result | (existing — none) | — | — |

Rounding lat/lng to 4 decimal places (~11m grid) is fine for cache keys; sub-11m differences in input still resolve to the same building.

---

## 10. Performance budget

### 10.1 Cold cache (worst case)

| Step | Worst case | Why |
|---|---|---|
| Coordinate parse | <5ms | Client-side regex |
| Microsoft quadkey fetch + parse | 10-25s | 30MB NDJSON over Azure Blob Storage on a cold quadkey |
| OSM Overpass building query | 2-4s | Hosted Overpass instances under load |
| Places `/nearby` | 0.5-1.5s | Google API + Worker overhead |
| CH `/search/companies` | 0.4-1s | Companies House API |
| **Total (Overpass MVP)** | **4-7s** | Acceptable |
| **Total (Microsoft cold)** | **12-28s** | Approaching Worker timeout — must cache aggressively |

### 10.2 Warm cache

All layers <1.5s when the quadkey, places result, and tenant CH search are cached. **<2s total** is the steady-state target.

### 10.3 User-facing implication

The pin-drop button should show a loading spinner with text that changes as legs complete:
1. "Finding building…" (first 1-25s)
2. "Finding tenant…" (next 1-2s, after building)
3. "Matching Companies House…" (next 0.5-1s)
4. Result rendered

If we report progress at each leg, the worst case feels measured rather than hung.

---

## 11. Costs to track

| Service | Cost | Cap |
|---|---|---|
| Microsoft Global ML Building Footprints | Free (ODbL) | None documented; polite use assumed |
| OSM Overpass (overpass-api.de) | Free (ODbL) | ~1 req/s sustained; bursting OK |
| Cloudflare R2 storage (footprint cache) | Free up to 10GB/month | UK dataset is <2GB raw, ~700MB gzipped |
| Cloudflare KV (places/overpass cache) | Free up to 1000 writes/day | Pin-drop generates 1-2 KV writes per query — well within tier |
| Google Places `/nearby` | Same budget as existing `/places` | **100/day shared cap** — pin drops eat into the same allowance as normal Search Company lookups |
| Companies House API | Free with key | 600 req/5min — never close to it |

**One real budget pressure: the 100/day Places cap.** A heavy pin-drop session (50 prospects) plus a normal Search Company session (50 prospects) = 100+ Places calls = cap hit. Worth instrumenting the Worker to log Places usage per day so we know when we're close.

---

## 12. Build order

Recommended sequence (estimated 5-7 hrs total):

1. **Coordinate parser** (45 min): Client-side DMS + decimal parser with range check. Test against Earth Pro clipboard output for both formats. Unit-test by pasting known coords for known buildings.
2. **Worker `/pin-drop` skeleton** (30 min): New route, accepts `{lat, lng}`, returns `{ building: null, places: null, ch: null }`. Just the envelope.
3. **Overpass integration** (90 min): Worker fetches OSM building polygon, parses geometry, runs polygon-area formula. Returns `{ found, areaM2, polygon, source: "osm-overpass" }`.
4. **Places /nearby sub-route** (60 min): Worker calls Google Places Nearby Search at 50m, retries at 150m if empty. Returns matches in existing shape.
5. **CH search wiring** (30 min): Worker passes top-match `displayName` through `_cleanCompanyName` and hits `/search/companies`. Returns the items array.
6. **Worker fan-out + caching** (60 min): R2 cache for Overpass responses (until we swap to Microsoft); KV cache for Places. Internal Promise.all for parallel building+places, then sequential CH.
7. **UI build** (90 min): Pin coordinates input, Drop pin button, three-section result panel with greyed-out partial states.
8. **Auto-attach wiring** (30 min): When CH returns exactly one match, click-equivalent of `pickLookupResult` to populate currentProspect. Otherwise show the same dp-lookup-row markup we already use.
9. **End-to-end test** (30 min): RITTAL-CSM Plymouth (known good), a multi-tenant retail park (ambiguous Places case), a brand-new build absent from OSM (no-building case), a field with no tenant.

Microsoft Footprints swap (Phase 2, future session, est. 3-4 hrs):
- Replace Overpass backend with Microsoft quadkey fetcher
- R2-cache quadkey files (30-day TTL)
- Pre-warm common UK quadkeys via a one-time cron job

---

## 13. Out of scope (for the first build)

- **Microsoft Building Footprints backend** — Phase 2, after the OSM MVP is proven end-to-end.
- **Sidewalk / boundary visualisation** — no map preview in v1; the user already has Earth Pro open with the polygon visible there.
- **Multi-pin prospecting** — single pin per session. Repeat the flow for each prospect.
- **Pin drop on the Manual form** — pin drop assumes the user has already opened the proposal card (Search-found, Manual-continued, or Saved-loaded). Adding it to the Manual tab is a small follow-up but not essential — most pin-drop flows start with "I've found a building and want to know who's there", which fits the Search → detail panel mental model.
- **Saving the polygon to Airtable** — the canonical area number is what we persist (existing `TotalRoofArea` field). The polygon itself is transient; Stage A's `currentProspect.location` already captures the centroid coordinates we need for the KML.
- **Microsoft Footprints rate-limit telemetry** — Microsoft hasn't documented one; if we hit issues post-launch we'll add monitoring then.
- **OSM Overpass attribution UI** — required by ODbL when displaying data publicly. Internal tool only; not user-facing today. Worth a footer credit in v2 if we ever expose this externally.

---

*Spec written Mon 25 May 2026. Source of truth for the Pin Drop Prospecting build. Update this doc if decisions change during build. Reference specs in repo: `managing-agent-phone-cadence.md` for the doc style template.*
