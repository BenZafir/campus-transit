# CampusBus — transit inside closed sites

Public transit systems stop at the gate. Campuses, industrial parks, ports and
large compounds run their own shuttles, and the people riding them have no idea
where the bus is or when it will arrive.

CampusBus is a multi-region system for running those shuttles: service areas as
geofenced polygons, stops, lines and routes, schedules, live GPS from drivers,
ETA, off-route detection, and an operations console — all configurable remotely,
with no app release required to change how a site operates.

**[▸ Open the interactive prototype](https://benzafir.github.io/campus-transit/)** · ~200 screens across three apps, running in the browser

---

## What is in this repository

| | |
|---|---|
| [`docs/spec.md`](docs/spec.md) | The full product and system specification — 22 sections: domain model, RBAC, trip lifecycle, map matching, event catalogue, NFRs, a canonical MVP feature list, and the open decisions still to be made. Written in Hebrew. |
| [`docs/adr/`](docs/adr/) | Architecture decision records — why each structural choice was made, and what it costs. |
| [`prototype/`](prototype/) | An interactive, data-driven prototype of all three surfaces plus the simulator. No backend; it runs entirely in the browser. |
| `src/` | The implementation. A vertical slice, not the whole spec — see *Scope* below. |

## The decisions worth arguing about

Every system has a handful of choices that shape everything after them. These
are the ones here, each with an ADR:

**There is no Vehicle entity.** A driver with an active trip *is* the bus, and
their phone's location is the bus's location. Fleet management is a different
product; modelling it here would have added a whole entity graph to serve no
user-visible behaviour.

**Regions may not overlap.** Overlapping service areas make "which region does
this stop belong to" ambiguous, and that ambiguity propagates into permissions,
ETA and analytics. The restriction is enforced geometrically at publish time.
It will be revisited only if a real business case appears.

**Configuration goes through Draft → Validate → Publish.** Operational config —
stops, routes, schedules — changes far more often than code. Letting an operator
edit production directly is how a site loses its bus service on a Tuesday
morning, so changes are validated against geometry and ordering rules before
they become visible to apps.

**Stale location is never shown as live.** Every position carries a server-side
timestamp, and the UI degrades through explicit freshness states — live, aging,
stale, no-signal — each visually distinct. A bus that stopped reporting looks
different from one that is parked.

**The simulator is in the MVP, not the backlog.** You cannot develop a
location-based system while depending on physically driving around a site. The
simulator runs a virtual driver along a route, at a chosen speed, with injectable
GPS loss and route deviation — so every behaviour in the spec is reachable from a
desk.

## Architecture

```
React Native (passenger)  ─┐
React Native (driver)     ─┼─ REST ──▶  ASP.NET Core API  ──▶  PostgreSQL + PostGIS
React + TS (admin)        ─┘                   │
                           └─ SignalR ─────────┘   geofencing, map matching,
                                                    off-route, ETA
```

Chosen for portability rather than for any one cloud: ASP.NET Core, PostgreSQL
with PostGIS, containers and an object-storage abstraction. `docs/spec.md §14.2`
compares Azure, AWS and GCP on managed-Postgres PostGIS support and monthly cost
at pilot and small-production scale, and deliberately does not pick one.

## Scope of the implementation

The specification describes a complete product. The code here is a **vertical
slice** through it — enough to prove the system works end to end, not enough to
run a site:

**In:** regions with geofencing · stops, lines, routes · one full trip lifecycle ·
driver GPS ingest over SignalR with offline buffering · map matching and ETA ·
off-route detection · passenger view · the simulator · docker-compose and CI

**Out, deliberately:** the admin CRUD screens (config is seeded; the prototype
shows the intended UI) · React Native clients, in favour of a responsive web
passenger view · authentication beyond a stub · analytics · feedback and
service requests

Where the code stops, the prototype and the spec continue. That boundary is
itself documented in `docs/adr/`.

## Running it

```sh
docker compose up --build
```

Then open the passenger view, start the simulator, and watch a bus move.

## A note on the scenario

The site, its regions, stops, routes, drivers and every number in the prototype
are invented. The prototype's map is schematic and contains no coordinates of
any kind.

## Licence

MIT.
