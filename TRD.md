# Technical Requirements Document — Smart Parking Navigator MVP

| Field        | Value                                         |
|--------------|-----------------------------------------------|
| Document     | TRD.md                                        |
| Version      | 1.0.0                                         |
| Status       | Draft                                         |
| Author       | GitHub Copilot (generated)                    |
| Created      | 2026-08-19                                    |
| Last Updated | 2026-08-19                                    |
| Closes Issue | #2                                            |
| Depends On   | PRD.md v1.0.0                                 |

---

## 1. Purpose

This document defines the technical architecture, component responsibilities,
data flows, API contracts, error-handling strategy, and test requirements for
the Smart Parking Navigator MVP. It is derived from `PRD.md` and is scoped to
the workshop deliverable only.

---

## 2. Solution Structure

The solution uses **.NET Aspire** and consists of four projects:

| Project                              | Role                                                               |
|--------------------------------------|--------------------------------------------------------------------|
| `CarparkAvailability.AppHost`        | Aspire orchestrator; wires up projects and injects connection info |
| `CarparkAvailability.ApiApp`         | ASP.NET Core Minimal API; owns all backend data access             |
| `CarparkAvailability.WebApp`         | Blazor Server web frontend; owns all UI and map rendering          |
| `CarparkAvailability.ServiceDefaults`| Shared Aspire service defaults (telemetry, health, resilience)     |

---

## 3. Component Responsibilities

### 3.1 AppHost (`CarparkAvailability.AppHost`)

- Declare `ApiApp` and `WebApp` as Aspire resources.
- Inject the `ApiApp` service URL into `WebApp` via `WithReference`.
- No application logic; no secrets.

### 3.2 ApiApp (`CarparkAvailability.ApiApp`)

- **HDB CSV ingestion** — load and parse the HDB Carpark Information CSV on
  startup; hold the parsed records in memory.
- **SVY21 conversion** — convert each car park's `x_coord` / `y_coord` (SVY21)
  to WGS84 latitude / longitude on ingestion.
- **data.gov.sg polling** — poll the Carpark Availability endpoint every 60 s
  using a background `IHostedService`; cache the latest response in memory.
- **Join** — merge the in-memory HDB static data with the latest live data on
  every API request (join key: `car_park_no` = `carpark_number`).
- **Distance calculation** — compute the Haversine distance (km) between each
  car park's WGS84 coordinates and the caller-supplied origin point; return the
  value as part of the response.
- **API endpoints** — expose the endpoints described in section 5.
- **data.gov.sg API key** — read from `ApiApp` configuration / environment
  variable `DATAGOVSG__APIKEY`; never sent to the frontend.

### 3.3 WebApp (`CarparkAvailability.WebApp`)

- Render the Blazor Server UI: search box, Google Maps embed, filter panel,
  results list, and detail panel.
- Call `ApiApp` endpoints via a typed `HttpClient`; never call data.gov.sg
  directly.
- **Google Maps API key** — read from `WebApp` configuration /
  environment variable `GOOGLEMAPS__APIKEY`; injected into the Razor page that
  loads the Maps JavaScript API. The key is visible in the browser but must not
  appear in `ApiApp`.
- Manage freshness state display (Fresh / Stale / Unavailable) based on the
  `fetchedAt` and `updateDatetime` fields returned by `ApiApp`.
- Poll `ApiApp` for updated data every 60 s using a Blazor timer or
  `PeriodicTimer`.

### 3.4 ServiceDefaults (`CarparkAvailability.ServiceDefaults`)

- Register OpenTelemetry tracing, metrics, and logging.
- Configure default health check endpoints (`/health`, `/alive`).
- Apply default `HttpClient` resilience policies (retry with exponential
  back-off, circuit breaker) via `Microsoft.Extensions.Http.Resilience`.
- Applied to both `ApiApp` and `WebApp` via `AddServiceDefaults()`.

---

## 4. Data Ingestion

### 4.1 HDB Carpark Information CSV

| Field             | Source column       | Type after parsing  | Notes                                          |
|-------------------|---------------------|---------------------|------------------------------------------------|
| `CarParkNo`       | `car_park_no`       | `string`            | Primary join key                               |
| `Address`         | `address`           | `string`            |                                                |
| `XCoord`          | `x_coord`           | `double`            | SVY21 Easting; converted to `Longitude`        |
| `YCoord`          | `y_coord`           | `double`            | SVY21 Northing; converted to `Latitude`        |
| `Latitude`        | _(derived)_         | `double`            | WGS84; computed from SVY21                     |
| `Longitude`       | _(derived)_         | `double`            | WGS84; computed from SVY21                     |
| `CarParkType`     | `car_park_type`     | `string`            | e.g. `MULTI-STOREY CAR PARK`                   |
| `TypeOfParkingSystem` | `type_of_parking_system` | `string`  | e.g. `ELECTRONIC PARKING`                     |
| `ShortTermParking`| `short_term_parking`| `string`            |                                                |
| `FreeParking`     | `free_parking`      | `string`            | `YES` / `NO` / timed value                    |
| `NightParking`    | `night_parking`     | `string`            | `YES` / `NO`                                   |
| `CarParkDecks`    | `car_park_decks`    | `int`               | Parsed from string                             |
| `GantryHeight`    | `gantry_height`     | `double`            | Metres; parsed from string                     |
| `CarParkBasement` | `car_park_basement` | `string`            | `Y` / `N`                                      |

Ingestion steps:

1. Read the CSV file from the path configured in `ApiApp` (`HDB__CsvPath`).
2. Skip the header row; parse each subsequent row.
3. Rows where `x_coord` or `y_coord` is zero or empty are stored without
   coordinates (excluded from distance-based queries).
4. Parse numeric string fields (`car_park_decks`, `gantry_height`) with
   `double.TryParse` / `int.TryParse`; default to `0` on failure.
5. Log a warning for each row that fails to parse.

### 4.2 SVY21 to WGS84 Conversion

SVY21 is a Transverse Mercator projection centred on Singapore. The conversion
formula used is the one published by the Singapore Land Authority (SLA).

Key constants:

| Symbol | Value                      | Description                     |
|--------|----------------------------|---------------------------------|
| a      | 6 378 137.0 m              | Semi-major axis (WGS84)         |
| f      | 1 / 298.257 223 563        | Flattening (WGS84)              |
| k₀     | 1.0                        | Central scale factor (SVY21)    |
| φ₀     | 1.366 666 rad (1° 22′ N)   | Origin latitude                 |
| λ₀     | 103.833 333° E             | Origin longitude                |
| N₀     | 38 744.572 m               | False Northing                  |
| E₀     | 28 001.642 m               | False Easting                   |

The conversion must be implemented as a pure static method with no external
dependencies. Unit tests must cover the reference point
(N 38 744.572, E 28 001.642) → (1.366 666°N, 103.833 333°E) and at least two
additional known Singapore locations.

### 4.3 data.gov.sg Polling

- Endpoint: `https://api.data.gov.sg/v1/transport/carpark-availability`
- Method: `GET`
- Authentication: `Authorization: <ApiKey>` request header using the value of `DATAGOVSG__APIKEY`.
- Polling interval: 60 s via `PeriodicTimer` inside an `IHostedService`.
- The hosted service stores the last successful `ApiResponse` and the
  `DateTimeOffset fetchedAt` timestamp in a thread-safe singleton
  `CarparkAvailabilityCache`.
- On polling failure the cache retains the previous value; the error is logged
  at `Warning` level.
- The cache exposes `IsStale` returning `true` when
  `DateTimeOffset.UtcNow - fetchedAt > TimeSpan.FromMinutes(5)`.

Response shape (from OpenAPI specification):

```json
{
  "items": [
    {
      "timestamp": "2024-01-15T10:30:00+08:00",
      "carpark_data": [
        {
          "carpark_number": "A1",
          "update_datetime": "2024-01-15T10:29:00+08:00",
          "carpark_info": [
            {
              "lot_type": "C",
              "total_lots": "100",
              "lots_available": "42"
            }
          ]
        }
      ]
    }
  ]
}
```

Notes:

- `total_lots` and `lots_available` are strings; convert with
  `int.TryParse`, default to `0` on failure (as specified in `IDEATION.md`).
- Accept backward-compatible additions (unknown optional fields).
- Treat missing required fields, incompatible types, or structural changes as
  schema validation errors; log at `Error` level and skip the affected record.

---

## 5. API Endpoints (ApiApp)

All endpoints are HTTP/1.1 + JSON. `ApiApp` registers `ServiceDefaults` and
exposes `/health` and `/alive`.

### 5.1 `GET /carparks`

Returns the joined dataset filtered to car parks within `radiusKm` of the
supplied origin.

**Query parameters:**

| Parameter    | Type   | Required | Default | Description                       |
|--------------|--------|----------|---------|-----------------------------------|
| `lat`        | double | Yes      | —       | Origin latitude (WGS84)           |
| `lng`        | double | Yes      | —       | Origin longitude (WGS84)          |
| `radiusKm`   | double | No       | 0.5     | Search radius in kilometres       |
| `availableOnly` | bool | No    | false   | Exclude car parks with 0 available lots |
| `freeParking` | bool  | No       | false   | Include only free-parking car parks |
| `nightParking` | bool | No      | false   | Include only night-parking car parks |
| `lotType`    | string | No       | —       | Filter by lot type: `C`, `H`, `M`, `Y` |

**Response body:**

```json
{
  "fetchedAt": "2024-01-15T02:30:00Z",
  "isStale": false,
  "items": [
    {
      "carParkNo": "A1",
      "address": "BLK 1 SAMPLE ROAD",
      "latitude": 1.3521,
      "longitude": 103.8198,
      "distanceKm": 0.12,
      "carParkType": "MULTI-STOREY CAR PARK",
      "typeOfParkingSystem": "ELECTRONIC PARKING",
      "shortTermParking": "7AM-10.30PM",
      "freeParking": "SUN & PH FR 1PM-10.30PM",
      "nightParking": "YES",
      "carParkDecks": 4,
      "gantryHeight": 2.1,
      "carParkBasement": "N",
      "updateDatetime": "2024-01-15T10:29:00+08:00",
      "lots": [
        { "lotType": "C", "totalLots": 100, "lotsAvailable": 42 }
      ]
    }
  ]
}
```

**Error responses:**

| HTTP Status | Condition                                     |
|-------------|-----------------------------------------------|
| 400         | `lat` or `lng` missing or out of range        |
| 503         | HDB CSV not yet loaded on startup             |

### 5.2 `GET /carparks/{carParkNo}`

Returns the full detail for a single car park.

**Path parameter:** `carParkNo` — the HDB car park number (e.g. `A1`).

**Response body:** Single item object from the shape above (without the
`fetchedAt` / `isStale` wrapper).

**Error responses:**

| HTTP Status | Condition               |
|-------------|-------------------------|
| 404         | Car park number unknown |

### 5.3 `GET /carparks/status`

Returns metadata about the last successful data.gov.sg poll.

```json
{
  "fetchedAt": "2024-01-15T02:30:00Z",
  "isStale": false,
  "totalCarParksInCache": 2010
}
```

---

## 6. Error Handling

| Failure Scenario                         | ApiApp Behaviour                                            | WebApp Behaviour                                       |
|------------------------------------------|-------------------------------------------------------------|--------------------------------------------------------|
| data.gov.sg returns non-2xx              | Log warning; retain previous cache value                   | Show last-fetched timestamp and "Live data unavailable" |
| data.gov.sg request times out            | Same as above                                               | Same as above                                          |
| data.gov.sg returns malformed JSON       | Log error; retain previous cache value                      | Same as above                                          |
| HDB CSV missing on startup               | Log critical; `GET /carparks` returns 503                   | Show "Service unavailable" banner                      |
| HDB CSV contains an unparseable row      | Log warning; skip row; continue ingestion                   | —                                                      |
| SVY21 conversion receives zero input     | Return `null` lat/lng; log warning                          | Exclude car park from map; include in list with no marker |
| ApiApp unreachable from WebApp           | — (resilience policy retries × 3 with back-off)            | After retries exhausted: show error toast               |

---

## 7. Credential Boundaries

| Credential              | Stored In                              | Used By              | Must NOT Appear In        |
|-------------------------|----------------------------------------|----------------------|---------------------------|
| Google Maps API key     | `WebApp` config / `GOOGLEMAPS__APIKEY` | WebApp Razor page    | ApiApp config, git history |
| data.gov.sg API key     | `ApiApp` config / `DATAGOVSG__APIKEY`  | ApiApp `HttpClient`  | WebApp config, browser HTML |

Both keys must be supplied via environment variables or user-secrets; they must
not be hard-coded in source files or committed to the repository.

---

## 8. Configuration Reference

### ApiApp (`appsettings.json` keys)

| Key                    | Type   | Default | Description                         |
|------------------------|--------|---------|-------------------------------------|
| `HDB:CsvPath`          | string | —       | Path to HDB Carpark Information CSV |
| `DataGovSg:ApiKey`     | string | —       | data.gov.sg API key                 |
| `DataGovSg:PollIntervalSeconds` | int | 60  | Polling interval                    |
| `DataGovSg:BaseUrl`    | string | `https://api.data.gov.sg` | API base URL      |
| `DataGovSg:StaleThresholdMinutes` | int | 5 | Minutes before cache is stale  |

### WebApp (`appsettings.json` keys)

| Key                    | Type   | Default | Description                         |
|------------------------|--------|---------|-------------------------------------|
| `GoogleMaps:ApiKey`    | string | —       | Google Maps JavaScript API key      |
| `ApiApp:BaseUrl`       | string | —       | Injected by AppHost via `WithReference` |
| `Polling:IntervalSeconds` | int | 60    | How often WebApp refreshes from ApiApp |

---

## 9. Test Requirements

### 9.1 Unit Tests

| Test                                                           | Component |
|----------------------------------------------------------------|-----------|
| SVY21 → WGS84: reference origin maps correctly                 | ApiApp    |
| SVY21 → WGS84: two additional known Singapore coordinates      | ApiApp    |
| SVY21 → WGS84: zero input returns null without throwing        | ApiApp    |
| HDB CSV parser: valid row produces correct model               | ApiApp    |
| HDB CSV parser: numeric string fields parse to correct types   | ApiApp    |
| HDB CSV parser: row with blank coords stores null lat/lng      | ApiApp    |
| Availability parser: `lots_available` string parses to int     | ApiApp    |
| Availability parser: missing field defaults to 0              | ApiApp    |
| `IsStale` returns `false` when `fetchedAt` < 5 min ago         | ApiApp    |
| `IsStale` returns `true` when `fetchedAt` > 5 min ago          | ApiApp    |
| Haversine distance: known distance between two coordinates     | ApiApp    |
| `GET /carparks` with missing `lat` returns 400                 | ApiApp    |
| `GET /carparks` filters `availableOnly` correctly              | ApiApp    |
| `GET /carparks/{id}` unknown id returns 404                    | ApiApp    |

### 9.2 Contract / Integration Tests

| Test                                                               | Notes                                      |
|--------------------------------------------------------------------|--------------------------------------------|
| `GET /carparks` returns 200 with valid response shape             | Uses `WebApplicationFactory`               |
| `GET /carparks/status` returns 200 with metadata shape             | Uses `WebApplicationFactory`               |
| Live data.gov.sg response validates against expected JSON schema   | Optional; skip in CI if key unavailable    |

### 9.3 Out of Scope for Tests

- End-to-end browser automation (Playwright) — deferred.
- Load / performance tests — deferred.
- Google Maps rendering — tested manually.

---

## 10. Out of Scope (Technical)

| Item                        | Reason                                         |
|-----------------------------|------------------------------------------------|
| Database / persistent storage | No storage infrastructure in MVP             |
| User authentication          | No identity system in MVP                    |
| Deployment pipeline          | Out of workshop scope                        |
| MCP integration              | Deferred to workshop step 05 capstone        |
| Agentic AI                   | Deferred to workshop step 05 capstone        |
| Non-HDB car parks            | Dataset limited to HDB inventory             |
| Private car park data        | Not available via data.gov.sg                |

---

## 11. Open Technical Questions

| # | Question                                                              | Owner    |
|---|-----------------------------------------------------------------------|----------|
| 1 | Should the HDB CSV be embedded as a build resource or read from disk? | Workshop |
| 2 | Is a concurrency-safe `ConcurrentDictionary` sufficient for the cache, or is `IMemoryCache` preferred? | Workshop |
| 3 | Should the Haversine calculation live in a shared library or directly in `ApiApp`? | Workshop |

---

## 12. Document History

| Version | Date       | Author                     | Change Summary |
|---------|------------|----------------------------|----------------|
| 1.0.0   | 2026-08-19 | GitHub Copilot (generated) | Initial draft  |
