# Data licences

The program in this repository is AGPL-3.0-only. The rail data is not. This note summarises three Rail Data Marketplace agreements the owner holds. The agreements themselves are not in the repository.

The standard data-sharing terms, used for Darwin and for fares, say the licence is non-exclusive and non-transferable (clause 2.2) and that the consumer may not transfer the agreement (clause 8.2). If the terms, Schedule 1, and a special condition conflict, clause 8.8 ranks them in that order: the terms, then Schedule 1 excluding special conditions, then the special conditions. Publishing must not suggest that the publisher endorses the publisher of this feed (clause 3.3.1).

## Darwin Real Time Train Information (Push)

Product `P-3f10bf96-d8e8-4041-aa5e-d75d82c45c4e`. Publisher: Rail Delivery Group. This is the Darwin agreement whose Schedule 1 was read. It is not the same product as Darwin Real Time Train Information v2.0 (`P-d3bf124c-1058-4040-8a62-87181a877d59`), which the owner also subscribed to. v2.0's signed schedule was not in the pack, so the rows below are not claimed for v2.0.

| | |
| --- | --- |
| Fee | Open. No charge. No fair-usage cap. |
| Updates | Live |
| Permitted purpose | Use, copy, cleanse, adapt, and aggregate. The new dataset may be distributed to third parties, **but not for commercial gain**. |
| Special condition | The data may not be resold as-is or substantially unmodified. Commercial use is permitted only where the data has been materially augmented, integrated with other data, or otherwise enhanced. |
| Precedence | The permitted purpose is Schedule 1. The commercial-use sentence is a special condition. Clause 8.8 ranks the special condition last, so it does not override "not for commercial gain". A free adapted feed fits the purpose. Charging for the feed is not treated as allowed. |
| Attribution | When the data is published, attribute **National Rail**. |
| Territory | Global |
| Term | One year, renewing automatically on the same terms. The consumer may end it on one week's notice. |
| Retention | Data already received may be kept after the term. |

Forecast times in the GTFS-Realtime feed are still copied rather than amended. That is a separate rule from the older NRE portal terms, and this feed keeps it. Raw topic payloads and snapshot bodies are not served.

## Darwin Real Time Train Information v2.0

Product `P-d3bf124c-1058-4040-8a62-87181a877d59`. The owner subscribed to this as well. It is Kafka, with a JSON topic and an AVRO topic, and the publisher says the inner schema is not fully documented. No signed Schedule 1 for it was in the pack. A subscribe-page summary that mentioned distribution and a ban on resale as-is is not a substitute for that schedule, and it did not contain the "not for commercial gain" sentence above.

v2.0 stays the live feed this program reads. It is the only Darwin subscription that names a JSON encoding and downloadable schemas (`darwin-base-schema.json` and `darwin-json-and-avro-schema.json`). The Push agreement names the product and the rights, and it does not name a wire format, so the parser is not switched onto it. Until a signed v2.0 schedule is read, the published feed also stays inside the stricter Push rules: free to fetch, attributed to National Rail, raw messages withheld, and not sold.

## NWR Track Model

Product `P-9b4e960e-8bb6-438b-9722-34ae5768a48f`. Publisher: Network Rail. The agreement opens with the Open Government Licence v3.0 and then a Schedule 1.

The OGL text grants a worldwide, royalty-free, perpetual licence to copy, publish, distribute, transmit, adapt, and exploit the information commercially and non-commercially. If the publisher gives no attribution statement, the OGL requires: "Contains public sector information licensed under the Open Government Licence v3.0."

Schedule 1 is narrower, and it is the part that matches the signed product:

| | |
| --- | --- |
| Fee | Open (OGL3). No charge. |
| Updates | Monthly |
| Permitted purpose | The raw data may be made freely available or otherwise distributed. |
| Attribution | No organisation is named in the schedule. |
| Territory | **UK only.** No prohibited countries are listed. |
| Term | One year. The consumer may end it on one week's notice. The schedule does not say the year renews automatically. |
| Retention | Data already received may be kept. |
| Special conditions | None. |

The OGL's "worldwide" grant and the schedule's "UK" restriction are both in the same agreement. This design does not treat the OGL sentence as permission to ignore the schedule.

**Open question.** The owner's Raspberry Pi is in Boston, USA. Schedule 1 permits exercise of the licensed rights in the UK. It is unresolved whether reading, transforming, or serving this product on infrastructure outside the UK is inside that territory. Do not download or process the Track Model until the owner answers that.

If the territory allows it, the model is a later source of track and platform geometry. A VehiclePosition could then carry a coordinate taken from the model, and a platform stop could be placed on the track, instead of publishing no position at all. It is not an input to v1.

## Fares, full refresh, weekly

Product `P-03ed70e9-cc9d-45f3-949b-45adffc70791`. Publisher: Rail Delivery Group. Same style of agreement as the Darwin Push product.

| | |
| --- | --- |
| Fee | Open. No charge. No fair-usage cap. |
| Updates | Weekly |
| Permitted purpose | Use, copy, cleanse, adapt, and aggregate. Distribution to third parties, **but not for commercial gain**. |
| Special conditions | None. There is no "materially augmented" exception in this agreement. |
| Attribution | No organisation is named in the schedule. Clause 3.3.1 still requires credit to the publisher, Rail Delivery Group, without implying endorsement. |
| Territory | Global |
| Term | One year, renewing automatically. The consumer may end it on one month's notice. |
| Retention | Data already received may be kept. |

This is a possible later input to GTFS Fares v2. It is not part of v1, and it does not block the realtime feed.

## Not in this pack

Darwin Timetable Files (`P-9ca6bc7e-62e1-44d6-b93a-1616f7d2caf8`), the snapshots product, CORPUS Location, and Knowledgebase Stations were not among these three agreements. The owner has said the timetable files are under OGL 3 and that they are not subscribed yet. That statement is not re-checked here.
