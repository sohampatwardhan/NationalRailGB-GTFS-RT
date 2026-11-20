# AGENTS.md

Instructions for coding agents working in this repository.

## Project

National Rail GB GTFS-RT (AGPL-3.0-only) will publish GTFS-Realtime for Great Britain's national rail network from the Darwin Push Port, and the static GTFS that feed binds to. The design is `docs/superpowers/specs/2026-11-20-darwin-gtfs-rt-design.md`. The implementation plan is `docs/superpowers/plans/2026-11-20-darwin-gtfs-rt.md`. Gaps between Darwin and GTFS are `docs/gtfs-gaps.md`.

The repository does not yet contain a crate, a container, or a workflow. Do not invent a second architecture beside the design. Do not start implementation unless the owner asks for a plan task.

The model is the owner's [Amtrak-GTFS-RT](https://github.com/sohampatwardhan/Amtrak-GTFS-RT): one Rust process, one container, GHCR images for amd64 and arm64, a Raspberry Pi as the deploy target, immutable generations, a peer allowlist, and the MobilityData validator in CI. Copy those conventions. Do not copy Amtrak's sources.

## Documentation

All documentation uses British English. Write **licence** for the noun and **license** for the verb. Write organisation, colour, centre, timetabled, modelling, behaviour, cancelled, and labelled. Keep GTFS and GTFS-Realtime identifiers as published. The realtime enum is `CANCELED`; do not rename it in code or in prose that quotes the spec.

Any print or PDF output, and any page-size setting in documentation tooling, uses ISO A4. Do not set US Letter or any other ANSI size. There is no PDF toolchain in the repository. If one is added, its default page is A4 (`papersize: a4`, or the tool's equivalent). Pandoc, mdbook, typst, and CSS `@page` are included.

## How this feed differs from Amtrak

- The live source is Darwin Real Time Train Information v2.0 on Kafka (JSON topic). Static schedules are Darwin Timetable Files. Resync is the snapshots REST product (hourly Push Port v18). Stops come from CORPUS Location and Knowledgebase Stations. It is not the legacy STOMP broker, and it is not scraped.
- There is no official static GTFS to download. This process builds the zip. `trip_id` is the Darwin RID.
- There is no browser worker, no Chromium, and no Python service. Do not add one.
- The data licence is the subscribed RTTI v2.0 Open licence (distribution allowed, no resale as-is, commercial use only when materially enhanced) and OGL 3 for the timetable files. Forecast times are copied, not amended. Raw topic and snapshot bodies are never served. Kafka bootstrap, consumer key, consumer secret, and group id are secrets and are never committed or logged.
- Associations, platform confirmation, "delayed" with no estimate, formations, and passing points are specified in the design and in `docs/gtfs-gaps.md`. Do not encode a Join or a Divide as `block_id` or as `transfer_type` 4.
- One image, `ghcr.io/sohampatwardhan/national-rail-gb-gtfs-rt`. No role environment variable, no second image, no extra feed URLs.

## Target layout

When implementation starts, the plan creates these paths. Do not create them early.

| Path | Role |
| --- | --- |
| `src/main.rs` | Config, ingest, builder, HTTP |
| `src/config.rs` | `from_map`, peer allowlist, secret-free `Debug` |
| `src/store.rs` | In-memory Darwin copy |
| `src/static_gtfs.rs` | Zip builder |
| `src/trip_updates.rs` | TripUpdates |
| `src/vehicle_positions.rs` | VehiclePositions |
| `src/alerts.rs` | Alerts |
| `src/associations.rs` | Sidecar and `transfers.txt` rows |
| `src/serve.rs` | axum, ETag, peer check |
| `fixtures/` | Offline JSON messages and a small v18 snapshot |
| `Dockerfile`, `docker-compose.yml` | One image, Pi-shaped compose |
| `.github/workflows/` | CI, release, Dependabot |

HTTP paths, environment variables, and the generation rules are in the design. Do not add root-level `/trip-updates.pb`, `/vehicle-positions.pb`, `/alerts.pb`, or `/static.zip`.

## Setup, once the crate exists

The plan's toolchain is Rust stable (the image builder is `rust:1.96-bookworm`), `protoc`, and a JRE 17 for the validator. Unit tests do not need Java, Docker, or a Darwin login.

Offline checks the plan requires:

```bash
cargo fmt --all -- --check
cargo test --locked
cargo clippy --locked --all-targets --all-features -- -D warnings
```

A live Darwin connection is an ignored test. Do not un-ignore it to make CI call the Rail Data Marketplace.

## Coding conventions

Match Amtrak-GTFS-RT where the design does not say otherwise.

- Rust edition 2021. rustfmt defaults. No `rustfmt.toml`.
- Small `Display` error types. No `anyhow` or `thiserror` on the config and publish path.
- `Config::from_map` so tests inject a map.
- `Debug` for config reports only whether a secret is set.
- Document public items. A non-trivial module opens with a `//!` comment that states the invariant.
- Silence a lint with `#[expect(clippy::..., reason = "...")]`, not a blanket `allow`.
- New behaviour needs an offline test.

## What agents must not do

- Do not commit secrets, `.env` files, or the Kafka bootstrap, consumer key, consumer secret, or group id.
- Do not commit raw topic payloads or hourly snapshots from a live feed. Fixtures are small and synthetic.
- Do not log credentials, raw messages, or URLs that contain secrets.
- Do not serve the generation directory with a generic file server.
- Do not treat `X-Forwarded-For` or `Forwarded` as the client.
- Do not amend a Darwin forecast, publish a working forecast, or invent a lat/lon, a platform, or a delay.
- Do not publish a deactivated, deleted, or suppressed service.
- Do not copy `catenarytransit/nationalrail-gtfs` (no licence file) or any other repository that has no licence.
- Do not file issues or pull requests on `google/transit`. The gap note records the proposals; the owner decides whether to file them.
- Do not push tags or publish to GHCR unless the owner asks.
- Do not weaken the validator digest check to make a build pass.

## Pull requests and commits

- Branch from `main`. Commit subjects are imperative sentences with a period.
- Do not rewrite published history.
- A change to the feed, the config, or the image is recorded under `## [Unreleased]` in `CHANGELOG.md` once that file exists. The plan adds it with the crate.
- This documentation pass was committed on `main` because the owner asked for spec and steering docs without a pull request. Implementation tasks go through pull requests.

## GitHub issues

When you create or triage an issue, fill the fields GitHub offers where they fit. Leave a field empty rather than invent a value.

- Assignee: `sohampatwardhan`, unless told otherwise.
- Type: exactly one. **Task** is a specific piece of work. **Bug** is an unexpected problem. **Feature** is a request or new functionality. Do not create labels to stand in for the type.
- Labels: the repository defaults only (`bug`, `documentation`, `duplicate`, `enhancement`, `good first issue`, `help wanted`, `invalid`, `question`, `wontfix`). Do not create labels. Dependabot may add `dependencies` and an ecosystem label; those are allowed on its pull requests.
- Projects and milestones: do not set them.
- Relationships: use a parent or sub-issue for real containment, and blocked-by for real ordering. Set them with the GraphQL mutations `addSubIssue` and `addBlockedBy`, not with a sentence in the body.
- Development: link the pull request with a closing keyword (`Closes #N`) in its body.
