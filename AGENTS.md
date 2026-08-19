# Repository Instructions for Coding Agents

This document gives coding agents accurate context about the **Smart Parking Navigator** repository so they can work safely and consistently without inventing architecture, commands, or features that do not exist yet.

---

## Project overview

Smart Parking Navigator is a workshop project that will eventually let users find available HDB carparks near a location in Singapore. It is built on .NET Aspire and the solution is in the starter-scaffold state: the projects compile and run, but the parking-specific features have not been implemented yet.

---

## Repository layout

```
CarparkAvailability.slnx            Solution file
Directory.Packages.props            Central package version management
global.json                         Pins the .NET SDK version
src/
  CarparkAvailability.AppHost/      Aspire orchestrator – wires ApiApp and WebApp, injects secrets
  CarparkAvailability.ApiApp/       ASP.NET Core Minimal-API backend – calls data.gov.sg server-side
  CarparkAvailability.WebApp/       Blazor Server frontend – renders the map using Google Maps
  CarparkAvailability.ServiceDefaults/ Shared Aspire defaults (OpenTelemetry, health checks, resilience)
data/
  HDBCarparkInformation.csv         Static HDB carpark reference dataset (linked into ApiApp output)
  CarparkAvailability.json          Sample full API response from data.gov.sg
  carpark-availability-sample.json  Smaller sample API response
  carpark-availability.http         REST Client scratch file for manual API testing
docs/                               Step-by-step workshop guides
```

---

## Technology stack

| Concern | Technology |
|---|---|
| Runtime | .NET 10 (see `global.json`) |
| Orchestration | .NET Aspire 13 (`Aspire.AppHost.Sdk` 13.4.6) |
| Backend | ASP.NET Core Minimal API |
| Frontend | Blazor Server (`AddRazorComponents` + `AddInteractiveServerComponents`) |
| Shared defaults | `CarparkAvailability.ServiceDefaults` – OpenTelemetry, service discovery, HTTP resilience |
| Package management | Central Package Versioning (`Directory.Packages.props`) |

---

## Commands

All commands must be run from the repository root unless stated otherwise.

### Restore dependencies
```bash
dotnet restore CarparkAvailability.slnx
```

### Build (Release)
```bash
dotnet build CarparkAvailability.slnx --no-restore --configuration Release
```

### Run the full application (requires .NET Aspire workload)
```bash
dotnet run --project src/CarparkAvailability.AppHost
```

The Aspire dashboard is opened automatically. ApiApp and WebApp are started and wired together.

### Validate the build (same as CI)
```bash
dotnet restore CarparkAvailability.slnx
dotnet build CarparkAvailability.slnx --no-restore --configuration Release
```

There are no automated test projects in the solution yet. Do not invent test commands.

---

## Secrets and service boundaries

- **data.gov.sg API key** – injected into **ApiApp** only via the `DataGovSg__ApiKey` environment variable. The browser must never call data.gov.sg directly.
- **Google Maps API key** – injected into **WebApp** only via the `GoogleMaps__ApiKey` environment variable. It is used client-side in the browser for map rendering.
- Both keys are declared as Aspire secret parameters in `AppHost.cs` and sourced from `appsettings.json` or user secrets (`UserSecretsId: smart-parking-navigator-workshop-apphost`).
- **Never commit credentials, API keys, or secrets** to source control. Use `dotnet user-secrets` for local development.

---

## General guidelines

- Do not add packages that are not already present in `Directory.Packages.props`; add the `<PackageVersion>` entry there first, then the `<PackageReference>` in the project file.
- Do not call data.gov.sg from the WebApp or from any browser-executed code.
- Do not implement application features that are not described in the current PRD or TRD (once those documents exist).
- Keep all four projects inside `src/`; do not create new top-level source folders.
- Follow the existing coding style enforced by `.editorconfig`.

---

## Tests and documentation

- When tests are added they must live in a `tests/` folder at the repository root and be included in `CarparkAvailability.slnx`.
- Test coverage and test commands must follow the eventual PRD and TRD; do not invent test infrastructure before those documents are written.
- Documentation updates that describe new or changed behaviour must accompany the relevant code change.

---

## Commits and pull requests

- Use concise, imperative commit messages, e.g. `feat: add carpark search endpoint`.
- One logical change per commit.
- Pull requests must include a description of what changed and why.
- Fill in the PR template (`.github/PULL_REQUEST_TEMPLATE.md`) before requesting review.
- All CI checks (restore + build) must pass before merging.
- Do not merge your own pull request; request at least one review.
