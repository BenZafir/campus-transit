# CampusBus

Transit management for closed sites — campuses, industrial parks, ports. Three
surfaces (passenger, driver, admin) plus a simulator, over one .NET API.

`docs/spec.md` is the source of truth for behaviour. Read the relevant section
before implementing anything; if the code and the spec disagree, raise it rather
than silently picking one.

## Where things are

```
docs/spec.md        product + system specification (Hebrew), 22 sections
docs/adr/           architecture decision records
prototype/          browser-only design prototype, no backend — do not modify
src/Api/            ASP.NET Core API
src/Simulator/      virtual driver that drives a route
src/web/            responsive passenger view (React + TS)
```

`prototype/` is a finished artefact. It is reference for intended UI and
component vocabulary. Never edit it as part of implementing a feature.

## Stack

ASP.NET Core (.NET 10) · PostgreSQL + PostGIS · SignalR for realtime ·
React + TypeScript · Docker Compose locally · GitHub Actions for CI.

## The rules that must not be broken

These come from the spec and are the point of the project. If a change would
break one, stop and raise it.

1. **No Vehicle entity.** A driver with an active trip is the bus. Do not add
   vehicles, plates or fleet concepts. (`spec §2.2`, ADR-0001)
2. **Regions never overlap.** Enforce geometrically with PostGIS at publish
   time, not in application code after the fact. (`spec §5.1`, ADR-0002)
3. **Stale location is never presented as live.** Every position carries a
   server timestamp; the API exposes freshness, the UI renders the four states
   distinctly. Never let a client infer freshness from arrival time.
   (`spec §17`, ADR-0004)
4. **Off-route needs persistence, not a single sample.** A deviation requires a
   distance threshold sustained over time or samples. One bad GPS point is never
   an exception. (`spec §10.2`)
5. **Config changes go Draft → Validate → Publish.** Nothing an operator edits
   reaches apps without passing validation. (`spec §8.1`, ADR-0003)
6. **Every threshold is remote configuration.** GPS frequency, geofence grace,
   off-route distance and duration, ETA parameters — all read from config, never
   a constant in code. (`spec §MVP-21`)

## Conventions

- **Geospatial work belongs in PostGIS**, not in C#. Point-in-polygon,
  intersection, distance along a line — write the query.
- **Times are UTC** at every layer including the database. Convert at the edge.
- **Integration tests use Testcontainers** with real Postgres+PostGIS. Geofencing
  and map matching are exactly the things an in-memory fake would let through
  broken.
- **The simulator is a first-class client.** It speaks the same API as a real
  driver app — no special-case endpoints. Simulated trips are flagged in data
  and rendered distinctly, never mixed into analytics.
- One vertical slice per branch, PR into `main`. Conventional commits.
- Decisions go in `docs/adr/NNNN-short-title.md` — context, decision,
  consequences. I write these; you may review and challenge them.

## Build order

Keep it runnable at the end of every slice.

1. Domain + migrations: Region, Stop, Line, Route, Trip. PostGIS geometry
   columns. Seed one site from a JSON fixture.
2. Region validation: non-overlap, stops within bounds, route ordering.
3. Trip lifecycle: scheduled → active → completed, one active trip per driver.
4. GPS ingest over SignalR + offline buffer replay + freshness states.
5. Map matching: project a sample onto the route, compute progress.
6. ETA from live progress plus historical segment times.
7. Off-route detection with sustained-deviation rules.
8. The simulator.
9. Passenger web view: map, stop detail, arrivals.
10. docker-compose, CI, live demo deployment.

## Scope discipline

The spec describes a whole product; this repository implements a slice. The
README lists what is deliberately out — admin CRUD screens, React Native
clients, real auth, analytics, feedback. Do not start building those. If
something feels missing, it is probably out of scope on purpose; ask.

## Content rules

The site, regions, stops, routes and drivers are invented and must stay that
way. No real place names, no coordinates, no organisation names anywhere —
code, fixtures, comments, commit messages, ADRs or documentation.
