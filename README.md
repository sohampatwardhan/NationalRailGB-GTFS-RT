# NationalRailGB-GTFS-RT

Great Britain's National Rail Darwin realtime, published as GTFS-Realtime, with the static GTFS that feed binds to.

The service is not built yet. The design, the implementation plan, and the places Darwin does not fit GTFS are:

- [Design](docs/superpowers/specs/2026-11-20-darwin-gtfs-rt-design.md)
- [Implementation plan](docs/superpowers/plans/2026-11-20-darwin-gtfs-rt.md)
- [GTFS gaps](docs/gtfs-gaps.md)

Darwin covers schedules, forecasts, platforms, cancellations, and associations (splits, joins, stock links, and passenger connections) for the national network. The process will hold a copy of that database and publish one generation at a time: a static zip, TripUpdates, VehiclePositions, Alerts, and an associations sidecar. The stack follows [Amtrak-GTFS-RT](https://github.com/sohampatwardhan/Amtrak-GTFS-RT): Rust, one container image on GHCR for amd64 and arm64, and a Raspberry Pi as the deploy target. The sources differ. Darwin is an authenticated push from the [Rail Data Marketplace](https://www.nationalrail.co.uk/developers/darwin-data-feeds/), and National Rail does not publish a static GTFS file to pair it with.

## Licence

The code in this repository is [AGPL-3.0-only](LICENSE).

Darwin data is not. It is available to registered users under the [NRE Open Government Licence variation](https://opendata.nationalrail.co.uk/terms) and the Rail Data Marketplace product terms. Push feeds are free of the web-service usage cap. The terms allow publishing an adapted format, require National Rail attribution, and require that public forecasts stay consistent with Darwin. They do not allow the access credentials to be shared, or the raw feed to be resold unmodified. This program will copy public forecast times, attribute National Rail, and will not serve the raw XML.

## Prior art

No open-source Push Port to GTFS-Realtime converter was found. [gb-transit](https://github.com/planarnetwork/gb-transit) and [nationalrail-gtfs](https://github.com/catenarytransit/nationalrail-gtfs) build static GTFS from the CIF timetable. Their trip ids are not Darwin RIDs, so they are not the static half of this feed. Departure-board libraries and the RDM sample Kafka client stop at the transport.

## Status

Documentation only. Implementation follows the plan, one task at a time, and has not started.
