# Product Requirements Document — Smart Parking Navigator MVP

| Field        | Value                                         |
|--------------|-----------------------------------------------|
| Document     | PRD.md                                        |
| Version      | 1.0.0                                         |
| Status       | Draft                                         |
| Author       | GitHub Copilot (generated)                    |
| Created      | 2026-08-19                                    |
| Last Updated | 2026-08-19                                    |
| Closes Issue | #2                                            |

---

## 1. Purpose

This document defines the Minimum Viable Product (MVP) scope for the Smart
Parking Navigator workshop application. It captures the target users,
Singapore-specific context, user journeys, feature scope, explicit exclusions,
and measurable acceptance criteria. Implementation of application behaviour is
out of scope for this document.

---

## 2. Background and Context

### 2.1 Singapore Parking Landscape

Singapore's Housing & Development Board (HDB) operates more than 2,000
publicly accessible car parks across the island. The data.gov.sg platform
exposes two complementary datasets:

- **HDB Carpark Information** — static CSV listing address, car park type,
  parking system, free-parking hours, night-parking flag, height restrictions,
  and SVY21 coordinates for each car park.
- **Carpark Availability API** — live JSON feed updated approximately every
  minute, providing total and available lot counts per car park and lot type
  (`C` for car, `H` for heavy vehicle, `M` for motorcycle, `Y` for other).

Motorists currently have no single interface that combines static car park
details with real-time availability and distance to their intended destination.

### 2.2 Workshop Setting

This application is built as a hands-on learning exercise. The solution must
remain simple enough to be scaffolded and explained within a single workshop
session while demonstrating realistic patterns for .NET Aspire, Blazor, and
external API integration.

---

## 3. Target Users

| Persona           | Description                                                                                  |
|-------------------|----------------------------------------------------------------------------------------------|
| **Daily Commuter** | Drives to an HDB estate regularly; needs a quick answer for which car park has free lots.    |
| **Occasional Driver** | Visits an unfamiliar neighbourhood; needs to discover nearby HDB car parks at all.         |
| **Height-Restricted Vehicle Owner** | Drives a van or SUV; must avoid car parks with low deck clearance.        |

All personas are assumed to be on a modern smartphone browser or laptop in
Singapore with reliable internet connectivity.

---

## 4. User Journeys

### 4.1 Search by Destination

1. User opens the application.
2. User types a destination address or landmark into the search box.
3. The application geocodes the query using Google Maps and centres the map.
4. HDB car parks within a configurable radius (default 500 m) appear as map
   markers.
5. A sorted list of car parks appears in a side panel, ordered by distance.
6. User selects a car park to view its details and live lot counts.

### 4.2 Browse Nearby Car Parks

1. User grants browser location permission.
2. The application centres the map on the user's current location.
3. Nearby car parks appear as described in journey 4.1 from step 4 onward.

### 4.3 Filter Results

1. User applies one or more filters: available lots only, free parking, night
   parking, height limit, lot type.
2. The map and list update in real time to show only matching car parks.
3. User clears filters to restore the full result set.

### 4.4 View Car Park Details

1. User clicks a map marker or list item.
2. A details panel shows address, type, parking system, short-term hours, free
   parking conditions, night parking flag, number of decks, height restriction,
   underground status, and the live lot breakdown by type.
3. If the last `update_datetime` is more than five minutes in the past the
   panel shows a "Data may be stale" warning.
4. If no live data is available the panel shows the time of the last successful
   fetch and a "Live data unavailable" message.

---

## 5. MVP Feature Scope

### 5.1 In Scope

| # | Feature                              | Notes                                                   |
|---|--------------------------------------|---------------------------------------------------------|
| 1 | Destination search                   | Geocoding via Google Maps JavaScript API                |
| 2 | Nearby HDB car parks on map          | Markers within configurable radius                      |
| 3 | Live lot availability                | Polled from data.gov.sg every 60 s                      |
| 4 | Basic filters                        | Available only, free parking, night parking, lot type   |
| 5 | Car park detail panel                | All static fields plus live lot counts                  |
| 6 | Freshness states                     | Fresh / Stale / Unavailable clearly indicated           |

### 5.2 Out of Scope (MVP)

| Excluded Feature            | Reason                                                 |
|-----------------------------|--------------------------------------------------------|
| User accounts and favourites | Deferred to post-MVP                                  |
| Availability alerts          | Requires push infrastructure                          |
| Occupancy forecasting        | Requires historical data collection                   |
| Traffic / weather integration | Out of workshop scope                                |
| Height / vehicle-type profile | Deferred to post-MVP                                 |
| Operations dashboard         | Deferred to post-MVP                                  |
| Deployment to production     | Out of workshop scope                                 |
| Databases / persistent storage | No storage infrastructure in MVP                    |
| MCP (Model Context Protocol) | Deferred to workshop step 05 capstone                |
| Agentic AI                   | Deferred to workshop step 05 capstone                |

---

## 6. Measurable Acceptance Criteria

| ID   | Criterion                                                                                           |
|------|-----------------------------------------------------------------------------------------------------|
| AC-1 | Given a valid Singapore address, the map centres within 2 s and shows car park markers.             |
| AC-2 | Car parks are filtered to within 500 m of the searched destination by default.                      |
| AC-3 | Each map marker and list row shows available / total lot count.                                     |
| AC-4 | The lot counts on screen reflect an API response no older than 60 s under normal conditions.        |
| AC-5 | When `update_datetime` is > 5 min old the UI shows a "Data may be stale" warning.                  |
| AC-6 | When the API is unreachable the UI shows the timestamp of the last successful response.             |
| AC-7 | Applying the "Available only" filter removes all markers and list rows where available lots = 0.    |
| AC-8 | Applying the "Free parking" filter removes car parks where `free_parking` = "NO".                  |
| AC-9 | Applying the "Night parking" filter removes car parks where `night_parking` = "NO".                |
| AC-10 | Selecting a car park shows all static detail fields and the live lot breakdown.                    |
| AC-11 | At least 2,000 HDB car parks are loaded from the static dataset on startup.                        |
| AC-12 | The application builds and starts with `dotnet run --project src/CarparkAvailability.AppHost`.     |

---

## 7. Non-Functional Requirements

| Category       | Requirement                                                              |
|----------------|--------------------------------------------------------------------------|
| Performance    | Initial page load < 3 s on a 10 Mbps connection.                         |
| Availability   | Live data polling must not crash the application on repeated API errors. |
| Security       | Google Maps API key kept in WebApp config; data.gov.sg key in ApiApp config; neither key appears in client-side HTML. |
| Accessibility  | Map must have a keyboard-navigable alternative list view.                |
| Internationalisation | English only for MVP.                                             |

---

## 8. Open Questions

| # | Question                                              | Owner     |
|---|-------------------------------------------------------|-----------|
| 1 | Should the default search radius be user-adjustable?  | Workshop  |
| 2 | Is a 60 s polling interval acceptable to data.gov.sg? | Workshop  |

---

## 9. Document History

| Version | Date       | Author                  | Change Summary         |
|---------|------------|-------------------------|------------------------|
| 1.0.0   | 2026-08-19 | GitHub Copilot (generated) | Initial draft        |
