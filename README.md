# NationalRailGB-GTFS-RT

Great Britain's National Rail Darwin realtime, published as GTFS-Realtime, with the static GTFS that feed binds to.

The service is not built yet. The design, the implementation plan, and the places Darwin does not fit GTFS are:

- [Design](docs/superpowers/specs/2026-11-20-darwin-gtfs-rt-design.md)
- [Implementation plan](docs/superpowers/plans/2026-11-20-darwin-gtfs-rt.md)
- [GTFS gaps](docs/gtfs-gaps.md)

Darwin covers schedules, forecasts, platforms, cancellations, and associations (splits, joins, stock links, and passenger connections) for the national network. The process will hold a copy of that database and publish one generation at a time: a static zip, TripUpdates, VehiclePositions, Alerts, and an associations sidecar. The stack follows [Amtrak-GTFS-RT](https://github.com/sohampatwardhan/Amtrak-GTFS-RT): Rust, one container image on GHCR for amd64 and arm64, and a Raspberry Pi as the deploy target. The sources differ. Darwin is an authenticated push from the [Rail Data Marketplace](https://www.nationalrail.co.uk/developers/darwin-data-feeds/), and National Rail does not publish a static GTFS file to pair it with.

## Licence

The code in this repository is [AGPL-3.0-only](LICENSE).

Darwin data is not. The owner holds two Darwin subscriptions. The live feed is [Darwin Real Time Train Information v2.0](https://raildata.org.uk/dashboard/dataProduct/P-d3bf124c-1058-4040-8a62-87181a877d59/overview) (JSON on Kafka). The signed schedule that was read is for Darwin Real Time Train Information (Push), `P-3f10bf96-d8e8-4041-aa5e-d75d82c45c4e`: distribution is allowed **but not for commercial gain**, resale as-is is forbidden, and published data is attributed to National Rail. A special condition that allows commercial use of a materially augmented dataset ranks below that purpose. The feed is therefore free and is not sold. Details, the NWR Track Model, and fares are in [docs/licences.md](docs/licences.md). Public forecasts are copied unchanged. Raw messages are not served. Kafka credentials stay out of the repository.

## Prior art

No open-source Push Port to GTFS-Realtime converter was found. [gb-transit](https://github.com/planarnetwork/gb-transit) and [nationalrail-gtfs](https://github.com/catenarytransit/nationalrail-gtfs) build static GTFS from the CIF timetable. Their trip ids are not Darwin RIDs, so they are not the static half of this feed. Departure-board libraries and the RDM sample Kafka client stop at the transport.

## Status

Documentation only. Implementation follows the plan, one task at a time, and has not started.
