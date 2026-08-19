# Technical Requirements Document — Smart Parking Navigator

**Status:** Draft  
**Scope:** MVP defined in `IDEATION.md`  
**Deferred:** Favorites, alerts, forecasting, accounts, databases, traffic, weather integration, MCP, deployment

---

## 1. Architecture overview

The application follows the three-tier structure established by the Aspire scaffold.

```
Browser
  └─ WebApp (Blazor Server)
       └─ ApiApp (ASP.NET Core Minimal API)
            ├─ HDBCarparkInformation.csv  (static, embedded at build time)
            └─ data.gov.sg Carpark Availability API  (live, server-side only)
```

- **AppHost** orchestrates both projects, injects secrets, and wires service discovery.
- **ServiceDefaults** provides OpenTelemetry, health checks, HTTP resilience, and service discovery to both projects.
- The browser may only communicate with WebApp. WebApp calls ApiApp via Aspire service discovery. ApiApp calls data.gov.sg directly with the server-side key. The browser never calls data.gov.sg.

---

## 2. Static data — HDB Carpark Information

### 2.1 Source

`data/HDBCarparkInformation.csv` is linked into ApiApp's output directory at `Data/HDBCarparkInformation.csv`.

### 2.2 CSV columns

| Column | Type | Notes |
|---|---|---|
| `car_park_no` | string | Primary key; matches `carpark_number` in the live API |
| `address` | string | Human-readable street address |
| `x_coord` | decimal string | SVY21 easting — must be converted to WGS84 |
| `y_coord` | decimal string | SVY21 northing — must be converted to WGS84 |
| `car_park_type` | string | e.g. `MULTI-STOREY CAR PARK`, `BASEMENT CAR PARK`, `SURFACE CAR PARK` |
| `type_of_parking_system` | string | `ELECTRONIC PARKING` or `COUPON PARKING` |
| `short_term_parking` | string | e.g. `WHOLE DAY`, `7AM-10.30PM`, `NO` |
| `free_parking` | string | e.g. `NO`, `SUN & PH FR 7AM-10.30PM` — treat as opaque text; do not infer conditional logic |
| `night_parking` | string | `YES` or `NO` |
| `car_park_decks` | integer string | Number of decks; convert to integer |
| `gantry_height` | decimal string | Metres; convert to decimal |
| `car_park_basement` | string | `Y` or `N` |

### 2.3 Parsing rules

- Read the CSV at ApiApp startup and hold the result in memory.
- Skip rows where `car_park_no` is missing or empty; log a warning per skipped row.
- Treat numeric columns (`x_coord`, `y_coord`, `car_park_decks`, `gantry_height`) as strings in the source and parse them safely; treat parse failures as missing values rather than errors.
- Do not infer meaning for `free_parking` or `short_term_parking` values beyond exact-string comparison for filter purposes.

### 2.4 SVY21 → WGS84 coordinate conversion

ApiApp must convert each carpark's SVY21 `(x_coord, y_coord)` to WGS84 `(latitude, longitude)` using the standard SVY21 projection formula:

- Reference point: latitude 1° 22′ 02.9154″ N, longitude 103° 49′ 31.9752″ E
- False northing: 38 744.572 m, False easting: 28 001.642 m
- Scale factor: 1.0

Perform the conversion once at startup and store the results alongside the parsed CSV data. Do not call an external geocoding service for this conversion.

---

## 3. Live data — Carpark Availability API

### 3.1 Endpoint

```
GET https://api.data.gov.sg/v1/transport/carpark-availability
```

Authentication: `Authorization: ******` header.

### 3.2 Response structure

```json
{
  "items": [
    {
      "timestamp": "2026-08-10T21:29:37+08:00",
      "carpark_data": [
        {
          "carpark_number": "HE12",
          "update_datetime": "2026-08-10T21:28:38",
          "carpark_info": [
            {
              "lot_type": "C",
              "total_lots": "105",
              "lots_available": "30"
            }
          ]
        }
      ]
    }
  ]
}
```

### 3.3 Parsing rules

- Parse all string-typed numeric fields (`total_lots`, `lots_available`) to integers. Treat parse failures as missing.
- `update_datetime` has no timezone suffix; interpret it as Singapore Standard Time (UTC+8).
- `timestamp` includes a `+08:00` suffix; parse it as a `DateTimeOffset`.
- Multiple entries per `carpark_data` array per `lot_type` are valid; accumulate them.
- Accepted `lot_type` values: `C` (car), `H` (heavy vehicle), `Y` (motorcycle). Unknown values must be preserved but excluded from UI totals.

### 3.4 Polling and caching

- Poll the API at most once per minute per the data.gov.sg usage guidance.
- Cache the last successful response in memory on ApiApp.
- On a failed request, serve the cached response and include a `stale: true` flag and the timestamp of the last successful fetch in the API response.
- If no cached response exists and the request fails, return HTTP 503 with a problem-details body.

### 3.5 Matching with static data

- Join `carpark_number` (live) to `car_park_no` (static CSV) using exact, case-insensitive string comparison.
- If a live entry has no matching static record, include it in the response with the numeric availability data and null static fields.
- If a static record has no live entry, include it in the response with null availability data.

---

## 4. ApiApp endpoints

All endpoints are prefixed with `/api`. Responses use `application/json`.

### 4.1 `GET /api` — health/info

Returns name and status. Implemented in the starter scaffold.

### 4.2 `GET /api/carparks` — carpark list

Returns all carparks merged from static and live data.

**Query parameters:**

| Parameter | Type | Description |
|---|---|---|
| `lat` | double | WGS84 latitude of the search origin (required) |
| `lng` | double | WGS84 longitude of the search origin (required) |
| `radius` | int | Search radius in metres; default 500; maximum 2000 |
| `availableOnly` | bool | When true, exclude carparks with zero available lots of all types |
| `lotType` | string | Filter to a specific lot type (`C`, `H`, `Y`) |
| `nightParking` | bool | When true, include only carparks where `night_parking = YES` |
| `freeParking` | string | Exact-match filter on the `free_parking` field |
| `carParkType` | string | Exact-match filter on the `car_park_type` field |

**Response body (array of carpark objects):**

```json
{
  "carparkNo": "HE12",
  "address": "BLK 12 HE CARPARK",
  "latitude": 1.3521,
  "longitude": 103.8198,
  "distanceMetres": 213.4,
  "carParkType": "MULTI-STOREY CAR PARK",
  "typeOfParkingSystem": "ELECTRONIC PARKING",
  "shortTermParking": "WHOLE DAY",
  "freeParking": "NO",
  "nightParking": "YES",
  "carParkDecks": 5,
  "gantryHeight": 2.1,
  "carParkBasement": false,
  "availability": [
    {
      "lotType": "C",
      "totalLots": 105,
      "lotsAvailable": 30
    }
  ],
  "updateDatetime": "2026-08-10T21:28:38+08:00",
  "stale": false,
  "lastFetchedAt": "2026-08-10T21:29:37+08:00"
}
```

- `distanceMetres` is the Haversine distance from the supplied `lat`/`lng` to the carpark's WGS84 coordinates.
- `stale` is `true` if the live availability data is more than 5 minutes old at the time the request is served.
- `lastFetchedAt` reflects the `timestamp` field from the last successful data.gov.sg response.
- When no live data is available for a carpark, `availability` is an empty array and `updateDatetime` and `lastFetchedAt` are null.

### 4.3 `GET /api/carparks/{carparkNo}` — carpark detail

Returns a single merged carpark object by `carparkNo`. Returns HTTP 404 with a problem-details body if not found in the static dataset.

---

## 5. WebApp — Blazor Server frontend

### 5.1 Google Maps integration

- Load the Maps JavaScript API with the `GoogleMaps__ApiKey` environment variable rendered into the page by WebApp at request time.
- The key is sent to the browser as part of the Maps JS API script URL (`?key=...`) and will be visible in the browser. Restrict the key by HTTP referrer/origin as described in `docs/google-maps-api-key.md`.
- The key must not appear in server logs, error messages, or API responses.

### 5.2 Pages and components

| Component | Responsibility |
|---|---|
| `SearchBar` | Accepts destination text; uses Google Places Autocomplete restricted to Singapore |
| `MapView` | Renders Google Maps; displays carpark markers; updates as visible bounds change |
| `FilterPanel` | Exposes available-only, lot-type, night-parking, free-parking, and carpark-type filters |
| `CarparkList` | Sorted list of carparks within the search radius; links to `CarparkDetail` |
| `CarparkDetail` | Full static and availability data for one carpark |
| `StatusBanner` | Displays loading, empty, stale, unavailable, or error states |

### 5.3 State machine

The UI must display exactly one of these states at a time:

| State | Trigger | Display |
|---|---|---|
| `Idle` | Page load before search | Prompt user to search |
| `Loading` | Request in flight | Spinner or skeleton |
| `Results` | Successful response with ≥1 carpark | Map + list |
| `Empty` | Successful response with 0 carparks | "No carparks found" message |
| `Stale` | Response with `stale: true` | Results with staleness banner |
| `Unavailable` | ApiApp returns 503 | "Data temporarily unavailable" banner |
| `Error` | Network or unexpected error | Error message with retry option |

### 5.4 Refresh behaviour

- Re-fetch carparks when the user submits a new search or moves the map significantly (bounds change of more than the current search radius).
- Automatically re-fetch at most every 60 seconds while the page is active.
- Do not re-fetch while a request is already in flight.

---

## 6. Security

- `DataGovSg__ApiKey` is used only in ApiApp server-side HTTP requests. It must not appear in any client-side code, logs, error messages, or responses.
- `GoogleMaps__ApiKey` is used only in WebApp. Restrict the key to the Maps JavaScript API, Places API (New), and Geocoding API, and to the deployed origin(s).
- Use HTTPS for all external API calls.
- Validate and sanitise all user-supplied query-string parameters in ApiApp before use.
- Return `application/problem+json` for all error responses from ApiApp.
- No user data is persisted in this MVP. No authentication is required.

---

## 7. Accessibility

- All interactive elements must be keyboard-navigable.
- Map markers must have ARIA labels including the carpark address and available-lot count.
- Status banners must use ARIA live regions so screen readers announce changes.
- Colour must not be the sole means of conveying availability status.

---

## 8. Error handling

- ApiApp wraps all unhandled exceptions via `app.UseExceptionHandler()` and returns `application/problem+json`.
- WebApp catches `HttpRequestException` from calls to ApiApp and transitions to the `Error` or `Unavailable` state.
- Stale data is surfaced through the `stale` field; the UI must not silently display old data.
- Data.gov.sg API errors are logged at `Warning` level with the status code and a redacted request identifier. The API key must not be logged.

---

## 9. Testing

Test projects must be added to `tests/` at the repository root and referenced in `CarparkAvailability.slnx` when implemented. Specific test cases must be derived from the PRD acceptance criteria. Until the PRD is approved, do not invent test commands or test project names.

Expected coverage areas (non-exhaustive):

- SVY21 → WGS84 conversion with known reference points
- CSV parsing: well-formed rows, missing fields, unparseable numerics
- JSON deserialization of the data.gov.sg response shape
- Haversine distance calculation
- Endpoint contract tests for `/api/carparks` and `/api/carparks/{carparkNo}`
- Stale-data flag logic
- Frontend state transitions (idle → loading → results/empty/stale/error)

---

## 10. Deferred scope

The following items are explicitly out of scope for this MVP and must not be implemented:

- User accounts, profiles, or authentication
- Persistent storage or databases
- Favorites and availability alerts
- Historical data collection and occupancy forecasting
- Traffic or weather API integration
- Azure or cloud deployment
- MCP server
- AI parking agent (introduced separately in workshop step 05)
