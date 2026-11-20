# Darwin to GTFS-Realtime

Date: 2026-11-20

This is the design for [NationalRailGB-GTFS-RT](https://github.com/sohampatwardhan/NationalRailGB-GTFS-RT). It is a specification only. No service code is in the repository yet. The implementation plan is `docs/superpowers/plans/2026-11-20-darwin-gtfs-rt.md`. Gaps between Darwin and GTFS are `docs/gtfs-gaps.md`.

Documentation in this repository uses British English. GTFS identifiers stay as published, including the enum value `CANCELED`. Print, PDF, and any documentation page-size setting use A4.

## Problem

Apps that already consume GTFS-Realtime need Great Britain's national rail realtime: schedules, forecasts, platforms, cancellations, and associations from the Darwin Push Port. They also need a static GTFS feed whose trip and stop ids are the ones the realtime feed uses. National Rail does not publish that pair.

The owner's [Amtrak-GTFS-RT](https://github.com/sohampatwardhan/Amtrak-GTFS-RT) is the model for the stack and the way the feed is served: one Rust process, one container on GHCR for amd64 and arm64, a Raspberry Pi as the deploy target, a generation manifest, a peer allowlist, and a MobilityData validator in CI. The sources, the licence, and several GTFS mappings have to differ. Those differences are recorded below.

## Approaches

Three ways to produce the pair were considered.

1. **Poll the live departure board (OpenLDBWS or the JSON LDB).** Rejected. It is a per-station request API, the NRE usage charge starts above 5 million enquiries per railway period, and it does not carry the national association, formation, and schedule-update stream. Wrappers such as `nre-darwin-py` and Darwin.Net stop at the board.
2. **Build static GTFS from the CIF/DTD timetable, then attach realtime later.** Rejected as the published feed. [planarnetwork/gb-transit](https://github.com/planarnetwork/gb-transit) and [catenarytransit/nationalrail-gtfs](https://github.com/catenarytransit/nationalrail-gtfs) already do the static half. Their trip ids are CIF identities, not Darwin RIDs, so a Push Port TripUpdate would not join. `nationalrail-gtfs` has no `LICENSE` file; do not copy it.
3. **Hold a copy of the Darwin database and publish GTFS from that copy.** Chosen. The Darwin timetable, reference data, and snapshots are the static base. The Push Port is the update stream. One process publishes one generation: a static zip plus TripUpdates, VehiclePositions, and Alerts. This is the same shape as Amtrak-GTFS-RT's combined process, with a different source.

## Licensing and access

Two licence layers apply. The code in this repository is AGPL-3.0-only. The Darwin data is not.

The owner is subscribed, free, on an Open licence, to **Darwin Real Time Train Information v2.0** ([product](https://raildata.org.uk/dashboard/dataProduct/P-d3bf124c-1058-4040-8a62-87181a877d59/overview), id `P-d3bf124c-1058-4040-8a62-87181a877d59`). Access is Kafka pub/sub with a JSON topic and an AVRO topic. The subscribe page allows distribution, forbids reselling the data as-is, allows commercial use only where the data has been materially enhanced, and grants global use on a one-year term that renews automatically. A GTFS-Realtime feed is that enhancement. The raw JSON, AVRO, and snapshot bodies are not served.

The older NRE portal terms ([terms](https://opendata.nationalrail.co.uk/terms)) still describe the same push feeds: adapt the format, do not amend the content, attribute National Rail, and keep public forecasts consistent with Darwin. The [usage-charging document](https://assets.nationalrail.co.uk/e8xgegruud3g/1FMZaTCkitl5TXLHyEAEvw/a493521c9d9a4e90c012bfce5b16363a/Usage_Charging_Document_v06-01.pdf) (v06-01) puts no enquiry cap on push feeds. The legacy STOMP broker is not the access path for this project.

Kafka credentials are secrets and are never committed, logged, or put in an image:

- `NR_KAFKA_BOOTSTRAP`
- `NR_KAFKA_CONSUMER_KEY`
- `NR_KAFKA_CONSUMER_SECRET`
- `NR_KAFKA_GROUP_ID`

`NR_KAFKA_TOPIC` is the JSON topic name. It is configuration, not a secret, and it has no default. A missing bootstrap, key, secret, group id, or topic refuses to start. `Debug` prints `set` or `unset` only.

Consequences for the feed:

- Public forecast times are copied from Darwin. They are not smoothed, rounded differently, or replaced with a model.
- Working forecasts, which the Push Port spec says are operational and not for public display, are not published.
- Raw topic payloads and snapshot bodies are never served and never logged.
- `feed_info.txt` names National Rail as the source of the times and links to the National Rail developer page.
- Deactivated, deleted, and suppressed services are not published. The Push Port spec (P75301004, Issue 27, §3.3.1, published under [RSPS5051](https://www.rspaccreditation.org/publicDocumentation.php#RSPS5051)) says a deactivated service should not be published to the public.
- The HTTP API keeps the Amtrak peer allowlist so the Pi is not an open proxy by accident. That is an operational default, not a licence restriction: the subscribed terms allow distribution of the enhanced feed. Loopback only, unless `NR_ALLOWED_PEER_IPS` lists peers.

## Prior art

No open-source Darwin Push Port to GTFS-Realtime converter was found. Searches covered GitHub projects that mention the Push Port, GTFS-Realtime, and National Rail together, the Open Rail Data examples, and the RDM sample client.

| Project | What it does | How this project differs |
| --- | --- | --- |
| [planarnetwork/gb-transit](https://github.com/planarnetwork/gb-transit) | CIF/DTD to static GTFS, nightly, NaPTAN coordinates, passing points dropped by default (`cif2gtfs --remove-passing-points`) | No Push Port and no GTFS-Realtime. Trip ids are CIF identities. |
| [catenarytransit/nationalrail-gtfs](https://github.com/catenarytransit/nationalrail-gtfs) | Rust reader of the legacy NRDP CIF timetable. Writes static GTFS and a non-standard `associations.txt`. No licence file, no README. | No Push Port, no protobuf, no server. Different id space. Do not copy the code. |
| [raildatamarketplace/rdm-darwin-kafka-client](https://github.com/raildatamarketplace/rdm-darwin-kafka-client) | Sample consumer for the RDM Kafka topics | Transport only. |
| Open Rail Data STOMP examples, `nre-darwin-py`, Darwin.Net, arrivals-kmp | STOMP clients or LDB departure boards | They do not emit GTFS-Realtime. |

The live interface is Darwin RTTI v2.0 on Kafka. The JSON topic is the one this process consumes. The AVRO topic carries the same feed in another encoding and is not parsed in v1. The publisher says the inner JSON and AVRO schema is not fully documented. The contract to vendor, and not to invent, is the pair `darwin-base-schema.json` and `darwin-json-and-avro-schema.json` from that product's Documentation tab. Unknown fields are ignored. Where the schema is thin, field meaning comes from [RSPS5051](https://www.rspaccreditation.org/publicDocumentation.php#RSPS5051) (Darwin interface specifications, version 25, 11 August 2026, including P75301004 Issue 27) and from [RSPS5053](https://www.rspaccreditation.org/publicDocumentation.php#RSPS5053) (Darwin pub/sub via the Rail Data Marketplace, version 01-01, 31 March 2025).

Static schedules come from **Darwin Timetable Files** (`P-9ca6bc7e-62e1-44d6-b93a-1616f7d2caf8`), a FILE product under OGL 3. The files hold schedules at least 48 hours ahead, derived from the Network Rail CIF, and they are the base Darwin uses for RTTI. The owner is not subscribed to this product yet. There is no other full CIF product on the marketplace. The static task cannot run against production files until that subscription exists. The CIF readers in gb-transit and nationalrail-gtfs stay prior art, not a second timetable.

Resync uses **Darwin Real Time Train Information snapshots** (`P-8b01d5cf-2dd4-48b4-bb98-dc6d3520baf9`), a REST product of hourly Push Port v18 snapshots. The live topic is v2.0 JSON, so the snapshot parser and the JSON parser both emit `PushMessage` and do not share a decoder. A related Kafka product, Darwin Real Time Train Information (Push) v1.0 (`P-3f10bf96-d8e8-4041-aa5e-d75d82c45c4e`), is not the target.

Stops come from **CORPUS Location data** (`P-d84fe2ed-5fac-416e-ac5b-d9612311f93c`) for TIPLOC, CRS, and coordinates, and from **Knowledgebase Stations** (`P-88ffe920-471c-4fd9-8e0d-95d5b9b7a257`, JSON `P-9c97bd03-e2f2-462d-860a-5bec92700c2d`, spec [RSPS5050](https://www.rspaccreditation.org/publicDocumentation.php#RSPS5050)) for passenger names. NWR CORPUS (`P-4ae3e81c-7907-423b-940f-91ac6fdc69e1`) is the parent set; v1 reads the location slice, not the whole corpus. NWR BPLAN Geography is out of scope (no `shapes.txt` in v1). NaPTAN is not required.

## What is in the feed

One generation is one static GTFS zip and three protobuf products, plus one sidecar for associations. Consumers start at `GET /v1/feed-set.json` and then fetch the immutable URLs in that manifest.

| Product | Darwin source | GTFS target |
| --- | --- | --- |
| Static zip | Darwin Timetable Files, CORPUS Location, Knowledgebase Stations, confirmed platforms | `agency.txt`, `routes.txt`, `trips.txt`, `stop_times.txt`, `calendar_dates.txt`, `stops.txt`, `feed_info.txt`, and `transfers.txt` only for Next and Link |
| TripUpdates | TS forecasts and actuals, cancellations, platforms | One entity per published RID |
| VehiclePositions | Last reported public location, formation, loading | One entity per RID that has a reported location |
| Alerts | Cancellation reasons, late-running reasons, station messages, train alerts with passenger text | One entity per message |
| `associations.json` | Association elements | Sidecar, not inside the zip |

Out of the feed: empty coaching stock and any schedule with `isPassengerService` false, buses and ferries, working forecasts, operational alarms, passing points, unconfirmed platforms, suppressed platforms, deactivated and deleted schedules, and the raw topic and snapshot bodies.

## Architecture

```mermaid
flowchart LR
  subgraph rdm [Rail Data Marketplace]
    TT[Darwin Timetable Files]
    REF[CORPUS Location and Knowledgebase]
    SNAP[Snapshots REST, Push Port v18]
    BUS[RTTI v2.0 JSON topic]
  end
  subgraph proc [One process]
    IN[Ingest]
    ST[(Darwin store)]
    BLD[Generation builder]
    HTTP[axum]
  end
  TT --> IN
  REF --> IN
  SNAP --> IN
  BUS --> IN
  IN --> ST
  ST --> BLD
  BLD --> HTTP
  HTTP --> APP[GTFS consumers]
```

One container. No browser and no second process. Darwin is an authenticated push, unlike Amtrak's map feed, so the Chromium worker in Amtrak-GTFS-RT is not copied.

| Piece | Responsibility |
| --- | --- |
| `Config` | Environment via `from_map`. Secrets never appear in `Debug` or logs. |
| Ingest | Download the timetable, CORPUS Location, and Knowledgebase Stations. Pull the latest hourly snapshot over REST. Consume the JSON topic. Apply everything as `PushMessage`. On a gap the broker will not fill, resync from a snapshot. |
| `DarwinStore` | In-memory copy of activated passenger schedules, associations, forecasts, platforms, formations, loading, and passenger messages. |
| `StopCatalog` | TIPLOC, CRS, passenger name, and coordinate from CORPUS Location and Knowledgebase Stations. |
| Generation builder | Static zip, three protobuf feeds, associations sidecar. Pure function of the store, the catalogue, and a clock. |
| HTTP | Serve the last complete generation from memory. Never wait on Kafka or the RDM REST calls. |

Startup order: CORPUS Location and Knowledgebase Stations, then the timetable, then a snapshot, then the JSON topic. The first generation is published only when the catalogue, the timetable, and a snapshot have all been applied. A later failure leaves that generation in place. The AVRO topic is not read.

```mermaid
sequenceDiagram
  participant R as RDM files
  participant K as JSON topic
  participant S as Store
  participant G as Builder
  participant H as HTTP
  R->>S: CORPUS, Knowledgebase, timetable, v18 snapshot
  S->>G: First complete copy
  G->>H: Generation N
  K->>S: Schedule, association, TS, formation, loading, messages
  S->>G: Debounced rebuild
  G->>H: Generation N+1, or keep N if the build fails
```

Rebuilds are debounced by 5 seconds and skipped when the store has not changed. `/readyz` is 200 only while the served generation is younger than 300 seconds.

## Identity

These choices are deliberate. Changing them later breaks every consumer.

- `trip_id` is the Darwin RID, verbatim. `TripDescriptor.start_date` is the scheduled start date, `YYYYMMDD`.
- `service_id` is that same date. Service runs as `calendar_dates.txt` rows with `exception_type` 1. There is no weekly `calendar.txt`: a Darwin timetable is a short activated window, not a permanent rule.
- `trip_short_name` is the retail service id (`rsid`) when Darwin sends one, otherwise the headcode.
- `trip_headsign` is the public destination name.
- One `agency.txt` row per TOC code in the reference data. `agency_timezone` is `Europe/London`. `agency_url` is `https://www.nationalrail.co.uk`.
- `route_id` is the TOC code. `route_type` is 2. `route_long_name` is the TOC name. A national feed with one route per origin and destination pair is unstable and is not used.
- `feed_info.feed_version` is the timetable file id. When confirmed platform stops are added, the version gains `+plat.` and the first 8 hex digits of the platform table, and the realtime headers use that same version.

## Stops

`stop_id` for a scheduled calling point is the TIPLOC. `stop_code` is the CRS when the reference data maps it. Where several TIPLOCs share a CRS, a parent stop `crs:{CRS}` has `location_type` 1 and the TIPLOC stops are its children.

Coordinates come from CORPUS Location. Knowledgebase Stations supplies the passenger name when CORPUS has none. A passenger TIPLOC with no coordinate fails the static build. The previous generation keeps being served. Coordinates are not invented, and `0,0` is not written. Passing and operational TIPLOCs are not stops.

A platform stop `{tiploc}:plat:{label}` exists only for a platform Darwin has marked confirmed and not suppressed. `platform_code` is Darwin's label. `StopTimeUpdate.stop_time_properties.assigned_stop_id` points at it, and `stop_sequence` is always set in that case. An unconfirmed platform is not assigned. A suppressed platform (`platsup` true, which also covers CIS suppression and location suppression) is not assigned and is not a stop. This loses the "expected but not confirmed" state; see `docs/gtfs-gaps.md`.

## TripUpdates

For each published RID, one TripUpdate.

- A public forecast or actual arrival or departure is copied onto that stop's `StopTimeEvent`. Actual times already in the past use uncertainty 0. The time is Darwin's public time, not the working forecast.
- A cancelled service is `TripDescriptor.schedule_relationship = CANCELED`. Stop-level cancellations on a running service are `StopTimeUpdate.schedule_relationship = SKIPPED`.
- A location Darwin marks delayed, with no public estimate, gets `StopTimeUpdate.schedule_relationship = NO_DATA` and no arrival or departure. Other stops on the trip keep their own forecasts. No delay is invented. `NO_DATA` means "no realtime information", which is not what Darwin means; the gap is documented, and the encoding is the least wrong value that does not fabricate a minute count.
- A stop with neither a forecast, an actual, nor a delayed flag is omitted, so the static time stands.

## Associations

Darwin categories, from P75301004 §3.3.2: Join, Divide, Link, Next. Join and Divide: the main service continues through the TIPLOC and the associated service terminates or starts there. Link: passengers change trains. Next: the main service's stock forms the associated service.

`block_id` is the wrong tool for Join and Divide. The GTFS spec defines it as sequential trips of one vehicle, and says in-seat transfers belong in `transfer_type` 4. Two portions that run coupled over a shared section are not a sequence.

`transfer_type` 4 and 5 require the trips to meet end to end. The spec says the last stop of `from_trip_id` should be next to the first stop of `to_trip_id` ([reference](https://github.com/google/transit/blob/master/gtfs/spec/en/reference.md), [pull 303](https://github.com/google/transit/pull/303)). A Darwin divide, and Amtrak's Lake Shore Limited, share a middle section. Issue [548](https://github.com/google/transit/issues/548) asks to relax that for mid-trip attach and detach and suggests splitting into three trips. Splitting a RID would make the TripUpdate miss the service Darwin updates. This feed does not split RIDs.

Encoding that is published:

| Category | `transfers.txt` | Sidecar |
| --- | --- | --- |
| Next, and the TIPLOC is the last public stop of the main trip and the first public stop of the associated trip | `transfer_type` 5 | yes |
| Next that does not meet end to end | none | yes |
| Link | `transfer_type` 1 at that stop, both trip ids set | yes |
| Join, Divide | none | yes |
| Cancelled or deleted | omitted | omitted |

No `block_id` is written. The sidecar is `associations.json` beside the zip, schema `national-rail-associations/v1`, with `category`, `main_trip_id`, `associated_trip_id`, and `stop_id`. It is not a file inside the zip, so the GTFS validator does not see an unknown table. Detail and the Lake Shore Limited evidence are in `docs/gtfs-gaps.md`.

## VehiclePositions

Darwin does not provide a carriage latitude. No position is invented. A VehiclePosition is emitted only when a TS update has reported a public location for the RID. `current_stop_sequence` and `stop_id` are set. `current_status` is `STOPPED_AT` when the last report is an arrival with no later departure, and `IN_TRANSIT_TO` the next public stop otherwise. `vehicle.id` is the RID. `vehicle.label` is `trip_short_name`.

When a formation is present, `multi_carriage_details` lists every coach. `label` is the coach id. `carriage_sequence` is the order in the current direction of travel, which is how a reverse formation at a divide is expressed. `occupancy_percentage` is Darwin's 0–100 loading for that coach at that location. Loading does not ripple: a later stop keeps the last explicit value only in this process's reading of the formation, and a cleared formation publishes no percentages. Coach class, toilet state, and "detached from the front" have no field; they are gaps. `avgLoading` is a long-run average, not a realtime load, and is not copied into `occupancy_percentage`.

## Alerts

Alerts are rider-facing text Darwin has already written. They are not a second copy of every delay.

- A cancelled trip produces one alert. `effect` is `NO_SERVICE`. `informed_entity` is that trip and `start_date`. `header_text` and `cause_detail` are the reason wording from the reference data. `cause` stays `UNKNOWN_CAUSE`. A reviewed map from reason codes onto the GTFS cause enum is a later change, not part of the first build. Unknown wording is not forced into `TECHNICAL_PROBLEM`.
- A late-running reason produces one alert with `effect` `SIGNIFICANT_DELAYS` and the same text fields. The minute count stays in the TripUpdate.
- A station message or train alert that contains passenger text produces an alert with `effect` `OTHER_EFFECT` and that text as `header_text`. Operational alarms are dropped.
- A delayed flag with no estimate does not produce an alert.

## HTTP

Same contract as Amtrak-GTFS-RT, with `NR_` environment variables and an associations sidecar.

| Path | Access | Body |
| --- | --- | --- |
| `/livez` | public | Process is up |
| `/readyz` | peer | 200 only when the generation is under 300 seconds old |
| `/v1/feed-set.json` | peer | Generation id, timestamp, `feed_version`, entity counts, artefact URLs |
| `/v1/generations/{id}/static.zip` | peer | Static GTFS for that generation |
| `/v1/generations/{id}/trip-updates.pb` | peer | Protobuf |
| `/v1/generations/{id}/vehicle-positions.pb` | peer | Protobuf |
| `/v1/generations/{id}/alerts.pb` | peer | Protobuf |
| `/v1/generations/{id}/associations.json` | peer | Sidecar |

Responses carry `ETag` and `Last-Modified` and answer `If-None-Match` / `If-Modified-Since` with 304. `Forwarded` and `X-Forwarded-For` are ignored. An empty allowlist is loopback only. A non-loopback bind with no allowlist refuses to start. The container default allowlist is `gateway`, resolved to the container's IPv4 default gateway. 403 is an unauthorised peer, 404 is an unknown generation, 503 is no fresh generation.

There is no `/trip-updates.pb`, `/vehicle-positions.pb`, `/alerts.pb`, or `/static.zip` at the root.

## Errors

A bad message is logged with its type and RID and skipped. The store is not cleared. A failed timetable or snapshot download keeps the last good generation. A static build or validator error keeps the last good generation. A missing platform, a broken association, or a missing reason code never drops TripUpdates. Kafka reconnects with backoff. Credentials, URLs that contain them, and raw JSON or snapshot bodies are not logged. Unknown JSON fields are ignored, because the published schema is incomplete.

## Testing

Unit tests are offline. Config is injected with `from_map`. HTTP tests call the axum router with an injected peer and clock. Live-topic fixtures are small JSON documents checked into `fixtures/`. Snapshot fixtures are small Push Port v18 documents. A live Kafka or RDM test is `#[ignore]` and is not part of pull-request CI.

The CI fixture builds a tiny store (one trip, one divide, one cancellation, one confirmed platform, one formation) through the real builder. The MobilityData GTFS validator 8.0.1 must report zero errors on the zip. The MobilityData GTFS-Realtime validator must accept the three protobuf files. No Darwin host is called.

## Container and CI

Image `ghcr.io/sohampatwardhan/national-rail-gb-gtfs-rt`, `linux/amd64` and `linux/arm64`. Multi-stage build: `rust:1.96-bookworm` builder, runtime `debian:bookworm-slim` with tini and a JRE 17 for the validator. No Python and no Chromium. Compose publishes `127.0.0.1:8090:8080` and is enough to run on the Pi from the GHCR image without a checkout.

CI matches the Amtrak shape where it still applies: `cargo fmt`, tests, an offline validator fixture, native image builds on amd64 and arm64, and a docs-only skip so a change that touches only `*.md`, `docs/`, or `LICENSE` does not rebuild the image. The skip is per step, not a workflow-level path filter. Dependabot covers Cargo weekly, and GitHub Actions and Docker monthly, with the same grouping and cooldown as Amtrak-GTFS-RT. There is no pip ecosystem.

Release tags `v*.*.*` publish the one image. The release is not part of this design's implementation until the owner asks for a tag.

## What this design does not take from Amtrak

- No `getTrainsData`, no decrypting crate, no browser worker, no Akamai workaround.
- No official upstream static GTFS. This process builds the zip.
- No track-fetcher, Hartford board, or NJ Transit RailData.
- No second image and no `NR_ROLE`.
- The data licence is the RTTI v2.0 Open licence and, for the timetable, OGL 3. It is not a right to republish the raw topic.

## Open point

One fact blocks production static files, and it is not a design choice. The owner is subscribed to Darwin RTTI v2.0 and is not yet subscribed to Darwin Timetable Files. The snapshots REST product and CORPUS Location are part of this design and are not confirmed as active subscriptions. Until the timetable subscription exists, the static task can be built and tested on fixtures only.
