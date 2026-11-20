# Darwin to GTFS-Realtime Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish Great Britain's Darwin Push Port as a static GTFS zip plus GTFS-Realtime TripUpdates, VehiclePositions, and Alerts from one Rust process.

**Architecture:** One process downloads Darwin Timetable Files, CORPUS Location, and Knowledgebase Stations, pulls an hourly Push Port v18 snapshot over REST, consumes the Darwin RTTI v2.0 JSON topic, and keeps an in-memory store. A pure builder writes an immutable generation. axum serves the last good generation and never waits on the network.

**Tech Stack:** Rust edition 2021, axum, the `gtfs-realtime` crate, `prost` via that crate's build, Kafka for the JSON topic, the MobilityData GTFS validator CLI 8.0.1.

## Global Constraints

Copied from `docs/superpowers/specs/2026-11-20-darwin-gtfs-rt-design.md`. Every task includes these.

- Code licence is AGPL-3.0-only. Darwin RTTI v2.0 is the subscribed Open licence. Do not serve raw JSON, AVRO, or snapshot bodies. Do not commit `NR_KAFKA_BOOTSTRAP`, `NR_KAFKA_CONSUMER_KEY`, `NR_KAFKA_CONSUMER_SECRET`, or `NR_KAFKA_GROUP_ID`.
- Public forecast times are copied. Working forecasts are not published. No invented delay, platform, or coordinate.
- `trip_id` is the Darwin RID verbatim. `start_date` and `service_id` are the scheduled start date `YYYYMMDD`.
- `route_id` and `agency_id` are the TOC code. `route_type` is `2`. `agency_timezone` is `Europe/London`.
- `stop_id` for a call is the TIPLOC. Parent of several TIPLOCs that share a CRS is `crs:{CRS}`.
- Passing points are not `stop_times` rows. Deactivated, deleted, and suppressed services are absent.
- Join and Divide are not `block_id` and not `transfer_type` 4. The sidecar is `associations.json` outside the zip.
- Confirmed, non-suppressed platforms only. `assigned_stop_id` plus `stop_sequence`.
- Delayed with no estimate is `NO_DATA` on that stop only, with no alert and no invented minutes.
- Image `ghcr.io/sohampatwardhan/national-rail-gb-gtfs-rt`, `linux/amd64` and `linux/arm64`.
- Validator jar SHA-256 `19293ddd9b6f954f216d4f12054bd8a3232921751c4484339e339764a91000e2` (MobilityData GTFS validator CLI 8.0.1).
- Readiness window 300 seconds. Rebuild debounce 5 seconds.
- Host bind default `127.0.0.1:8080`. Empty peer allowlist means loopback only. Ignore `Forwarded` and `X-Forwarded-For`.
- Config comes from `from_map`. `Debug` and logs do not contain secrets or raw messages.
- Documentation added by a task uses British English. Any page size is A4.
- Unit tests are offline. A live Darwin test is `#[ignore]`.

---

## File structure

| File | Responsibility |
| --- | --- |
| `Cargo.toml` | Package `national-rail-gb-gtfs-rt`, edition 2021, one binary |
| `src/config.rs` | Environment |
| `src/model.rs` | `Service`, `Call`, `Association`, `Formation` |
| `src/store.rs` | Apply messages, snapshot replace, passenger query |
| `src/parse.rs` | RTTI v2.0 JSON, and a Push Port v18 snapshot, both to `PushMessage` |
| `src/static_gtfs.rs` | Zip bytes |
| `src/trip_updates.rs` | TripUpdate `FeedMessage` |
| `src/vehicle_positions.rs` | VehiclePosition `FeedMessage` |
| `src/alerts.rs` | Alert `FeedMessage` |
| `src/associations.rs` | Sidecar JSON and transfer rows |
| `src/generation.rs` | Assemble one generation from the store |
| `src/serve.rs` | Router |
| `src/main.rs` | Wire ingest, debounce, and HTTP |
| `fixtures/` | Synthetic JSON messages and a small v18 snapshot |
| `Dockerfile`, `docker-compose.yml` | One image, loopback publish |
| `.github/workflows/ci.yml` | fmt, test, validator fixture, images |
| `.github/dependabot.yml` | Cargo, Actions, Docker |

`src/model.rs` is created in Task 2 and only extended when a later task adds a field. Later tasks repeat the struct they rely on in their **Interfaces** block.

## Task 1: Service skeleton and generation HTTP API

**Files:**

- Create: `Cargo.toml`
- Create: `src/main.rs`
- Create: `src/config.rs`
- Create: `src/generation.rs`
- Create: `src/serve.rs`
- Create: `CHANGELOG.md`
- Test: `src/config.rs`, `src/serve.rs` (module tests)

**Interfaces:**

- Consumes: nothing
- Produces:
  - `config::Config::from_map(vars: &HashMap<String, String>) -> Result<Config, ConfigError>`
  - `generation::Generation { id, feed_version, created_unix, static_zip, trip_updates, vehicle_positions, alerts, associations }` all owned `String` or `Vec<u8>`
  - `serve::router(state: AppState) -> axum::Router`
  - `AppState` holds `Arc<RwLock<Option<Generation>>>`, the peer allowlist, and a clock trait `fn now_unix(&self) -> u64`

- [ ] **Step 1: Write the failing tests**

`src/config.rs` tests:

```rust
#[test]
fn loopback_bind_allows_an_empty_peer_list() {
    let cfg = Config::from_map(&map(&[
        ("NR_BIND", "127.0.0.1:8080"),
        ("NR_OUTPUT_DIR", "./out"),
    ]))
    .unwrap();
    assert!(cfg.allowed_peers.allows("127.0.0.1"));
    assert!(!cfg.allowed_peers.allows("10.0.0.8"));
}

#[test]
fn public_bind_without_peers_is_refused() {
    let err = Config::from_map(&map(&[("NR_BIND", "0.0.0.0:8080")]));
    assert!(err.is_err());
}

#[test]
fn debug_does_not_include_the_kafka_secret() {
    let cfg = Config::from_map(&map(&[
        ("NR_BIND", "127.0.0.1:8080"),
        ("NR_KAFKA_CONSUMER_SECRET", "s3cret-value"),
    ]))
    .unwrap();
    let text = format!("{cfg:?}");
    assert!(!text.contains("s3cret-value"));
    assert!(text.contains("set"));
}
```

`src/serve.rs` tests build a router with a fixed generation and call it with `tower::ServiceExt::oneshot`. Assert: `/livez` is 200 with no generation; `/readyz` is 503 when the generation is older than 300 seconds and 200 when it is younger; `/v1/feed-set.json` lists the five artefact paths; a second request with that `ETag` returns 304; a peer outside the allowlist gets 403; `X-Forwarded-For` does not change the peer.

- [ ] **Step 2: Run the tests**

Run: `cargo test --locked config:: serve::`

Expected: FAIL, modules missing.

- [ ] **Step 3: Implement the minimum**

Edition 2021 binary crate. Dependencies: `axum`, `tokio` (macros, rt-multi-thread), `serde`, `serde_json`, `sha2`. `ConfigError` and `PeerAllowlist` are small `Display` types in `config.rs`. `NR_ALLOWED_PEER_IPS=gateway` is accepted as a token and resolved in `main`, not in `from_map` (tests pass the literal `gateway` through and do not open a socket). `main` binds the address and serves `router`. Handlers read the `RwLock` and return. They do not fetch.

`feed-set.json` fields: `generation_id`, `feed_version`, `timestamp`, `static_url`, `trip_updates_url`, `vehicle_positions_url`, `alerts_url`, `associations_url`. URLs are paths under `/v1/generations/{id}/`.

- [ ] **Step 4: Run the tests**

Run: `cargo test --locked config:: serve::`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add Cargo.toml Cargo.lock src CHANGELOG.md
git commit -m "Add the generation HTTP API and peer allowlist."
```

`CHANGELOG.md` gains `## [Unreleased]` with one line for the HTTP API.

## Task 2: Darwin store and JSON apply

**Files:**

- Create: `src/model.rs`
- Create: `src/parse.rs`
- Create: `src/store.rs`
- Create: `fixtures/ts-delayed.json`
- Create: `fixtures/schedule-deactivated.json`
- Create: `fixtures/snapshot-v18.xml`
- Test: `src/store.rs`, `src/parse.rs`

**Interfaces:**

- Consumes: nothing from Task 1
- Produces:

```rust
pub struct Service {
    pub rid: String,
    pub start_date: String,
    pub toc: String,
    pub headcode: String,
    pub rsid: Option<String>,
    pub headsign: String,
    pub passenger: bool,
    pub cancelled: bool,
    pub deleted: bool,
    pub suppressed: bool,
    pub active: bool,
    pub cancel_reason: Option<String>,
    pub late_reason: Option<String>,
    pub calls: Vec<Call>,
}

pub struct Call {
    pub tiploc: String,
    pub public_arrival: Option<String>,
    pub public_departure: Option<String>,
    pub forecast_arrival: Option<String>,
    pub forecast_departure: Option<String>,
    pub working_arrival: Option<String>,
    pub actual_arrival: Option<String>,
    pub actual_departure: Option<String>,
    pub pass: bool,
    pub delayed_unknown: bool,
    pub cancelled: bool,
    pub platform: Option<Platform>,
}

pub struct Platform {
    pub label: String,
    pub confirmed: bool,
    pub suppressed: bool,
}

pub enum PushMessage {
    Schedule(Service),
    Deactivated { rid: String },
    Association(Association),
    Forecast(ForecastPatch),
    Formation(Formation),
    Loading(LoadingPatch),
    PassengerText(PassengerText),
}

impl DarwinStore {
    pub fn apply(&mut self, message: PushMessage);
    pub fn replace_all(&mut self, messages: Vec<PushMessage>);
    pub fn service(&self, rid: &str) -> Option<&Service>;
    pub fn published(&self) -> impl Iterator<Item = &Service>;
}
```

`published` yields services that are passenger, active, not deleted, and not suppressed. Forecast and actual times are the local `HH:MM` Darwin sent. `working_arrival` is stored and never read by later builders.

- [ ] **Step 1: Write the failing tests**

```rust
#[test]
fn deactivated_service_is_not_published() {
    let mut store = DarwinStore::new();
    store.apply(parse_fixture("fixtures/schedule-passenger.json"));
    store.apply(parse_fixture("fixtures/schedule-deactivated.json"));
    assert!(store.published().next().is_none());
}

#[test]
fn delayed_flag_does_not_invent_a_time() {
    let mut store = DarwinStore::new();
    store.apply(parse_fixture("fixtures/ts-delayed.json"));
    let call = &store.service("202601151234567").unwrap().calls[0];
    assert!(call.delayed_unknown);
    assert!(call.forecast_arrival.is_none());
}

#[test]
fn platsup_marks_the_platform_suppressed() {
    let mut store = DarwinStore::new();
    store.apply(parse_fixture("fixtures/ts-platform.json"));
    let platform = store.service("202601151234567").unwrap().calls[0]
        .platform
        .as_ref()
        .unwrap();
    assert!(platform.suppressed);
    assert_eq!(platform.label, "2");
}
```

Fixtures are synthetic and under 40 lines. `ts-delayed.json` is one train-status object with `delayed` true and no expected time on the public arrival. `ts-platform.json` sets the platform suppressed. `snapshot-v18.xml` is a minimal Push Port v18 snapshot that `parse_snapshot` turns into the same `PushMessage` values as the JSON fixtures. Include one association fixture used by Task 5, parsed here so the store test can round-trip `category`, both RIDs, the TIPLOC, and `cancelled`. Field names that the vendored schema does not define are not invented: the fixture uses names from `darwin-json-and-avro-schema.json` once that file is added, and the test fails if a required semantic (RID, delayed, platsup) is dropped.

- [ ] **Step 2: Run the tests**

Run: `cargo test --locked store:: parse::`

Expected: FAIL.

- [ ] **Step 3: Implement the minimum**

`parse.rs` accepts one RTTI v2.0 JSON object and returns one `PushMessage` or an error that names the object type. `parse_snapshot` accepts a Push Port v18 snapshot and returns the same enum. Unknown JSON fields are ignored. `apply` for `Deactivated` sets `active = false`. `apply` for `Forecast` patches calls by TIPLOC and the scheduled public time, so a circular call is not ambiguous. A missing service is kept as a forecast-only stub and `published` still hides it until a passenger schedule arrives. `replace_all` clears the store first. Log on a parse error: object type only.

- [ ] **Step 4: Run the tests**

Run: `cargo test --locked store:: parse::`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/model.rs src/parse.rs src/store.rs fixtures
git commit -m "Apply Darwin schedules, forecasts, and platforms to an in-memory store."
```

## Task 3: Static GTFS

**Files:**

- Create: `src/static_gtfs.rs`
- Create: `src/stops.rs`
- Create: `fixtures/corpus-location.csv`
- Test: `src/static_gtfs.rs`

**Interfaces:**

- Consumes: `DarwinStore::published`, `Service`, `Call` from Task 2
- Produces:

```rust
pub struct StopCatalog {
    pub by_tiploc: HashMap<String, StopRow>,
}

pub struct BuiltStatic {
    pub zip: Vec<u8>,
    pub feed_version: String,
    pub platform_stops: Vec<PlatformStop>,
}

pub fn build_static(
    store: &DarwinStore,
    stops: &StopCatalog,
    transfers: &[Transfer],
) -> Result<BuiltStatic, BuildError>;
```

`Transfer` is defined in Task 5. Until Task 5 lands, this task's tests pass `transfers: &[]`.

- [ ] **Step 1: Write the failing test**

```rust
#[test]
fn passenger_call_uses_the_rid_and_omits_a_passing_point() {
    let store = store_with_call_and_pass();
    let built = build_static(&store, &catalog_with_crs("YORK", "YORK"), &[]).unwrap();
    let files = unzip(&built.zip);
    let trips = &files["trips.txt"];
    assert!(trips.contains("202601151234567"));
    assert!(files["stop_times.txt"].contains("YORK"));
    assert!(!files["stop_times.txt"].contains("YORKPS"));
    assert!(!files["trips.txt"].contains("block_id"));
    assert_eq!(built.feed_version, "tt-test");
}

#[test]
fn missing_corpus_coordinate_fails_the_build() {
    let store = store_with_call_and_pass();
    let err = build_static(&store, &StopCatalog::default(), &[]);
    assert!(err.is_err());
}
```

`store_with_call_and_pass` builds a `DarwinStore` in code: one passenger service, RID `202601151234567`, start date `20260115`, TOC `GR`, one public call at `YORK`, one `pass` call at `YORKPS`.

- [ ] **Step 2: Run the test**

Run: `cargo test --locked static_gtfs::`

Expected: FAIL.

- [ ] **Step 3: Implement the minimum**

Write `agency.txt`, `routes.txt`, `trips.txt`, `stop_times.txt`, `calendar_dates.txt`, `stops.txt`, `feed_info.txt`. Add `transfers.txt` only when `transfers` is non-empty. `feed_publisher_name` is `National Rail Darwin via NationalRailGB-GTFS-RT`. `feed_publisher_url` is `https://www.nationalrail.co.uk/developers/darwin-data-feeds/`. `feed_version` argument comes from the timetable file id stored on the catalogue (`tt-test` in the unit test). Parent stops use `crs:{CRS}` when two published TIPLOCs share a CRS. `stop_lat` and `stop_lon` come from the catalogue. A published call whose TIPLOC has no coordinate returns `BuildError::MissingCoordinate { tiploc }`. Passing calls are skipped. `calendar_dates.txt` has one row per distinct `start_date`.

- [ ] **Step 4: Run the test**

Run: `cargo test --locked static_gtfs::`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/static_gtfs.rs src/stops.rs fixtures/corpus-location.csv
git commit -m "Build static GTFS from published Darwin schedules."
```

## Task 4: TripUpdates

**Files:**

- Create: `src/trip_updates.rs`
- Test: `src/trip_updates.rs`

**Interfaces:**

- Consumes: `Service`, `Call`, `Platform` from Task 2. `BuiltStatic::platform_stops` from Task 3.
- Produces: `pub fn build_trip_updates(store: &DarwinStore, feed_version: &str, now_unix: u64) -> FeedMessage`

- [ ] **Step 1: Write the failing tests**

```rust
#[test]
fn copies_the_public_forecast_and_does_not_copy_the_working_time() {
    let store = store_forecast("10:42", "10:40");
    let feed = build_trip_updates(&store, "tt-test", UNIX_2026_01_15_09_00_UTC);
    let event = arrival(&feed, "202601151234567", "YORK");
    assert_eq!(event.time, Some(UNIX_2026_01_15_10_42_UTC));
}

#[test]
fn delayed_without_an_estimate_is_no_data() {
    let store = store_delayed_unknown();
    let feed = build_trip_updates(&store, "tt-test", UNIX_2026_01_15_09_00_UTC);
    let update = stop_update(&feed, "202601151234567", "YORK");
    assert_eq!(update.schedule_relationship, Some(ScheduleRelationship::NoData));
    assert!(update.arrival.is_none());
}

#[test]
fn cancelled_trip_sets_the_trip_relationship() {
    let store = store_cancelled();
    let feed = build_trip_updates(&store, "tt-test", 0);
    assert_eq!(trip(&feed).schedule_relationship, Some(ScheduleRelationship::Canceled));
}

#[test]
fn confirmed_platform_sets_assigned_stop_id_and_sequence() {
    let store = store_platform(true, false, "4");
    let feed = build_trip_updates(&store, "tt-test", 0);
    let update = stop_update(&feed, "202601151234567", "YORK");
    let props = update.stop_time_properties.unwrap();
    assert_eq!(props.assigned_stop_id.as_deref(), Some("YORK:plat:4"));
    assert!(update.stop_sequence.is_some());
}

#[test]
fn suppressed_platform_is_not_assigned() {
    let store = store_platform(true, true, "4");
    let feed = build_trip_updates(&store, "tt-test", 0);
    assert!(stop_update(&feed, "202601151234567", "YORK").stop_time_properties.is_none());
}
```

`UNIX_2026_01_15_09_00_UTC` is `1768467600` and `UNIX_2026_01_15_10_42_UTC` is `1768473720`. 15 January 2026 10:42 in `Europe/London` is 10:42 UTC. The test date is in winter on purpose.

Helpers in the same module construct one published `Service` with RID `202601151234567`, start date `20260115`, and a single public call at `YORK`:

```rust
fn store_forecast(public_arrival: &str, working_arrival: &str) -> DarwinStore;
fn store_delayed_unknown() -> DarwinStore;
fn store_cancelled() -> DarwinStore;
fn store_platform(confirmed: bool, suppressed: bool, label: &str) -> DarwinStore;
fn arrival(feed: &FeedMessage, rid: &str, tiploc: &str) -> StopTimeEvent;
fn stop_update(feed: &FeedMessage, rid: &str, tiploc: &str) -> StopTimeUpdate;
fn trip(feed: &FeedMessage) -> TripDescriptor;
```

- [ ] **Step 2: Run the tests**

Run: `cargo test --locked trip_updates::`

Expected: FAIL.

- [ ] **Step 3: Implement the minimum**

One entity per `published` service. Header `gtfs_realtime_version` `2.0`, `incrementality` `FULL_DATASET`, `timestamp` `now_unix`, `feed_version` the argument. Convert local `HH:MM` with the `time` crate's `Europe/London` on `start_date`. Hours `24` and above roll the civil date. A cancelled service sets the trip relationship and no stop updates. A cancelled call on a running service is `SKIPPED`. An actual time in the past sets `uncertainty` to `0`. `vehicle.id` is the RID and `vehicle.label` is `rsid` or else `headcode`.

- [ ] **Step 4: Run the tests**

Run: `cargo test --locked trip_updates::`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/trip_updates.rs
git commit -m "Publish TripUpdates for forecasts, cancellations, and confirmed platforms."
```

## Task 5: Associations

**Files:**

- Create: `src/associations.rs`
- Modify: `src/static_gtfs.rs` (already accepts `&[Transfer]`)
- Test: `src/associations.rs`

**Interfaces:**

- Consumes: `Association` stored by Task 2 (`category`, `main_rid`, `assoc_rid`, `tiploc`, `cancelled`, `deleted`) and `Service::calls`
- Produces:

```rust
pub struct Transfer {
    pub from_trip_id: String,
    pub to_trip_id: String,
    pub from_stop_id: String,
    pub to_stop_id: String,
    pub transfer_type: u8,
}

pub fn associations_json(store: &DarwinStore) -> Vec<u8>;
pub fn transfers(store: &DarwinStore) -> Vec<Transfer>;
```

Sidecar schema `national-rail-associations/v1`: `{ "version", "associations": [ { "category", "main_trip_id", "associated_trip_id", "stop_id" } ] }`. Categories are lowercase `join`, `divide`, `link`, `next`.

- [ ] **Step 1: Write the failing tests**

```rust
#[test]
fn divide_is_sidecar_only() {
    let store = store_assoc("divide", "MAIN", "PORTION", "YORK");
    assert!(transfers(&store).is_empty());
    let json: serde_json::Value = serde_json::from_slice(&associations_json(&store)).unwrap();
    assert_eq!(json["associations"][0]["category"], "divide");
}

#[test]
fn next_meeting_end_to_end_is_transfer_type_5() {
    let store = store_next_at_boundary();
    let row = &transfers(&store)[0];
    assert_eq!(row.transfer_type, 5);
    assert_eq!(row.from_stop_id, "YORK");
}

#[test]
fn cancelled_association_is_omitted() {
    let store = store_assoc_cancelled();
    assert!(transfers(&store).is_empty());
    let json: serde_json::Value = serde_json::from_slice(&associations_json(&store)).unwrap();
    assert!(json["associations"].as_array().unwrap().is_empty());
}

#[test]
fn link_is_transfer_type_1() {
    let store = store_assoc("link", "MAIN", "ONWARD", "YORK");
    assert_eq!(transfers(&store)[0].transfer_type, 1);
}
```

`store_next_at_boundary` makes `YORK` the last public call of the main RID and the first public call of the associated RID. A second test, `next_in_the_middle_has_no_transfer`, asserts `transfers` is empty and the sidecar still has the row.

- [ ] **Step 2: Run the tests**

Run: `cargo test --locked associations::`

Expected: FAIL.

- [ ] **Step 3: Implement the minimum**

Cancelled or deleted associations are skipped. Join and Divide never produce a `Transfer`. Next produces `transfer_type` 5 only when the stop is the last public call of `main_rid` and the first public call of `assoc_rid`. Link always produces `transfer_type` 1. No function in this module sets `block_id`.

- [ ] **Step 4: Run the tests**

Run: `cargo test --locked associations:: static_gtfs::`

Expected: PASS. The static test with an empty transfer slice still has no `transfers.txt`. Add one assertion in `static_gtfs` that a single `Transfer` writes a `transfers.txt` row containing `5`.

- [ ] **Step 5: Commit**

```bash
git add src/associations.rs src/static_gtfs.rs
git commit -m "Publish Darwin associations without pretending they are blocks."
```

## Task 6: VehiclePositions

**Files:**

- Create: `src/vehicle_positions.rs`
- Test: `src/vehicle_positions.rs`

**Interfaces:**

- Consumes: `Service`, `Call`, and `Formation { rid, coaches: Vec<Coach> }` with `Coach { id, class: Option<String>, occupancy_percent: Option<i32> }`
- Produces: `pub fn build_vehicle_positions(store: &DarwinStore, feed_version: &str, now_unix: u64) -> FeedMessage`

- [ ] **Step 1: Write the failing tests**

```rust
#[test]
fn no_report_means_no_entity() {
    let store = store_with_schedule_only();
    let feed = build_vehicle_positions(&store, "tt-test", 0);
    assert!(feed.entity.is_empty());
}

#[test]
fn arrival_without_departure_is_stopped_at_and_has_no_lat_lon() {
    let store = store_actual_arrival("YORK");
    let pos = &build_vehicle_positions(&store, "tt-test", 0).entity[0].vehicle;
    assert!(pos.position.is_none());
    assert_eq!(pos.current_status, Some(VehicleStopStatus::StoppedAt));
    assert_eq!(pos.stop_id.as_deref(), Some("YORK"));
}

#[test]
fn loading_copies_the_percentage_and_does_not_put_class_in_the_label() {
    let store = store_formation("A", "First", 40);
    let carriage = &vehicle(&store).multi_carriage_details[0];
    assert_eq!(carriage.label.as_deref(), Some("A"));
    assert_eq!(carriage.occupancy_percentage, Some(40));
    assert_eq!(carriage.carriage_sequence, Some(1));
    assert_ne!(carriage.label.as_deref(), Some("First"));
}
```

The class assertion walks `multi_carriage_details` and checks there is no field whose debug text is `First`. The proto has no class field; the test fails if someone stuffs the class into `label`.

- [ ] **Step 2: Run the tests**

Run: `cargo test --locked vehicle_positions::`

Expected: FAIL.

- [ ] **Step 3: Implement the minimum**

Emit an entity only when some call has an actual arrival or departure. The current call is the latest actual. `STOPPED_AT` if that actual is an arrival and the same call has no actual departure. Otherwise `IN_TRANSIT_TO` the next public, non-pass call. Do not set `Position`. `carriage_sequence` follows the formation order. `label` is the coach id only. Set `occupancy_percentage` only when Darwin sent a number. Do not set `occupancy_status`.

- [ ] **Step 4: Run the tests**

Run: `cargo test --locked vehicle_positions::`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/vehicle_positions.rs
git commit -m "Publish VehiclePositions without inventing a coordinate."
```

## Task 7: Alerts

**Files:**

- Create: `src/alerts.rs`
- Test: `src/alerts.rs`

**Interfaces:**

- Consumes: `Service::cancel_reason`, `Service::late_reason`, `PassengerText { id, text, rid: Option<String> }`
- Produces: `pub fn build_alerts(store: &DarwinStore, feed_version: &str, now_unix: u64) -> FeedMessage`

- [ ] **Step 1: Write the failing tests**

```rust
#[test]
fn cancellation_reason_is_an_alert_and_not_a_rewritten_cause() {
    let store = store_cancelled_with_reason("points failure");
    let alert = &build_alerts(&store, "tt-test", 0).entity[0].alert;
    assert_eq!(alert.effect, Some(Effect::NoService));
    assert_eq!(alert.cause, Some(Cause::UnknownCause));
    assert_eq!(text(&alert.header_text), "points failure");
    assert_eq!(text(&alert.cause_detail), "points failure");
}

#[test]
fn delayed_unknown_does_not_create_an_alert() {
    let store = store_delayed_unknown();
    assert!(build_alerts(&store, "tt-test", 0).entity.is_empty());
}

#[test]
fn late_reason_is_significant_delays() {
    let store = store_late_reason("waiting for a late train");
    let alert = &build_alerts(&store, "tt-test", 0).entity[0].alert;
    assert_eq!(alert.effect, Some(Effect::SignificantDelays));
}

#[test]
fn empty_operational_alarm_is_dropped() {
    let store = store_passenger_text("");
    assert!(build_alerts(&store, "tt-test", 0).entity.is_empty());
}
```

- [ ] **Step 2: Run the tests**

Run: `cargo test --locked alerts::`

Expected: FAIL.

- [ ] **Step 3: Implement the minimum**

One alert per cancelled published service that has a reason string. One alert per late reason. One alert per passenger text whose trimmed body is non-empty, with `effect` `OTHER_EFFECT`. `informed_entity.trip.trip_id` is the RID and `start_date` is the service date when a RID is known. Do not map reason wording onto `TECHNICAL_PROBLEM` or any other non-unknown cause in this task. There is no cause map yet; adding one is a later change that needs its own fixture and a review note in `docs/gtfs-gaps.md`.

- [ ] **Step 4: Run the tests**

Run: `cargo test --locked alerts::`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/alerts.rs
git commit -m "Publish alerts from Darwin reason text and passenger messages."
```

## Task 8: Generation wiring, container, CI, validator, Dependabot

**Files:**

- Create: `src/ingest.rs`
- Modify: `src/main.rs`
- Modify: `src/generation.rs`
- Create: `Dockerfile`
- Create: `docker-compose.yml`
- Create: `.github/workflows/ci.yml`
- Create: `.github/dependabot.yml`
- Create: `scripts/validate-fixture.sh`
- Test: `src/generation.rs`

**Interfaces:**

- Consumes: `build_static`, `transfers`, `associations_json`, `build_trip_updates`, `build_vehicle_positions`, `build_alerts`, `serve::router`
- Produces:

```rust
pub fn assemble(store: &DarwinStore, stops: &StopCatalog, timetable_id: &str, now_unix: u64)
    -> Result<Generation, BuildError>;
```

`Generation.id` is `{timetable_id}-{now_unix}`. A platform overlay appends `+plat.` and the first 8 hex digits of the sorted platform-stop ids to `feed_version`.

- [ ] **Step 1: Write the failing test**

```rust
#[test]
fn assemble_rejects_a_missing_coordinate_and_returns_every_artefact() {
    let good = assemble(&store_with_call_and_pass(), &catalog_with_crs("YORK", "YORK"), "tt-test", 1_700_000_000).unwrap();
    assert!(!good.static_zip.is_empty());
    assert!(!good.trip_updates.is_empty());
    assert!(!good.associations.is_empty());
    let bad = assemble(&store_with_call_and_pass(), &StopCatalog::default(), "tt-test", 1);
    assert!(bad.is_err());
}
```

- [ ] **Step 2: Run the test**

Run: `cargo test --locked generation::`

Expected: FAIL.

- [ ] **Step 3: Implement the minimum**

`assemble` calls the five builders. `main` spawns ingest and a debounced rebuild: wait 5 seconds after the last successful `apply`, then `assemble`. On `Err`, keep the previous `Generation` in the `RwLock`. Ingest downloads CORPUS Location, Knowledgebase Stations, and the Darwin timetable, then the latest hourly snapshot, before subscribing to the JSON topic. The AVRO topic is not consumed. Kafka reconnect uses exponential backoff starting at 1 second and capped at 60. A snapshot replace happens when the client is told the subscription gap cannot be recovered, and also once at startup. `Dockerfile` is multi-stage: `rust:1.96-bookworm` builder, `debian:bookworm-slim` runtime, tini as PID 1, JRE 17, the validator jar checked against the SHA-256 in Global Constraints during image build. Compose publishes `127.0.0.1:8090:8080` and sets `NR_ALLOWED_PEER_IPS=gateway`.

`ci.yml` jobs: `fmt` (`cargo fmt --all -- --check`), `test` (`cargo test --locked`), `validator` (run `scripts/validate-fixture.sh`, which builds a fixture generation and runs validator 8.0.1 and the MobilityData GTFS-Realtime validator; the realtime validator release tag is pinned in `tools/gtfs-realtime-validator.txt` and the script exits 2 if that file is missing), `image-amd64`, `image-arm64`. A step that compiles or builds an image is skipped when `scripts/ci-scope.sh` says every changed file is documentation (`*.md`, `docs/`, `LICENSE`). The skip is inside the step, not a workflow `paths` filter. `.github/dependabot.yml` matches Amtrak-GTFS-RT for Cargo (weekly, patch cooldown 3, minor 7, major 30, grouped minor/patch and major, open limit 2), GitHub Actions (monthly, cooldown 7), and Docker (monthly, cooldown 7, patch-only for `rust` and `debian`). No pip ecosystem.

`scripts/validate-fixture.sh` must not read `NR_KAFKA_PASSWORD`.

- [ ] **Step 4: Run the tests**

Run: `cargo test --locked && cargo fmt --all -- --check`

Expected: PASS. The image jobs run in CI, not on a machine without Docker. Do not un-ignore any live test.

- [ ] **Step 5: Commit**

```bash
git add src/ingest.rs src/main.rs src/generation.rs Dockerfile docker-compose.yml .github scripts tools
git commit -m "Wire the generation builder, the container, and CI."
```

## Self-review

- Spec coverage: licensing and the "do not amend forecasts" rule are Global Constraints and Tasks 4 and 6. Identity, stops, and passing points are Task 3. TripUpdates, platforms, and `NO_DATA` are Task 4. Associations are Task 5. VehiclePositions and loading are Task 6. Alerts are Task 7. HTTP, peers, container, validator, and Dependabot are Tasks 1 and 8. The two owner questions are not tasks.
- Placeholders: none. The realtime validator tag is pinned in a file the script requires, not left as a guess in this plan.
- Types: `Service`, `Call`, `Platform`, `Generation`, `Transfer`, and `build_*` names match across tasks.
