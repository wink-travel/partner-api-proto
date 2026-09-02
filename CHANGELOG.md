# Changelog

All notable changes to the Partner API protobuf contracts are documented here. Format loosely
follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow
[Semantic Versioning](https://semver.org/) as described in [README.md](README.md#versioning--release-process).

Entries below `## [Unreleased]` are populated automatically by `publishPartnerProtoContracts.bash`
in `monorepo-java` when it cuts a release; do not hand-edit past entries.

## [Unreleased]

## [1.0.0] — 2026-09-02

Released from `monorepo-java` 55caee3152e0.

- fix(grpc-proto): :bug: make buf lint pass on the Partner proto release path (#837)
- feat(partner)!: GeoShape search area, city/agency identifiers, and financial-reporting fixes
- feat(partner)!: MIME-typed media formats and honest OTA coding (#826)
- fix(partner): take refund out of the booking status enum (#824)
- fix(booking): split the refund axis out of bookingStatus (#811)
- fix(inventory): media pruning crash, hero images, commission rounding, and OpenAPI operation names (#790)
- feat(partner): add account discovery to the gRPC partner API (#783)
- feat(partner): collapse the rate-calendar cache key to seven bounded components (#754)
- feat(partner): :sparkles: implement the Travel Agent Booking gRPC service (2/3)
- feat(partner): :sparkles: publish the Travel Agent Booking gRPC contract (1/3)
- feat(partner)!: :sparkles: split Content into GetProperties (batch) and GetProperty (single)
- feat(partner): :sparkles: conditional content fetch via cache_token and NOT_MODIFIED (#728)
- feat(booking)!: :boom: collapse Itinerary to a single RoomConfiguration (#724)
- feat(partner)!: :boom: price transactional inventory, and rework RoomRate money to unit/total (#723)
- feat(partner)!: :boom: replace Media with the Content API, meter every surface, one billing product
- feat(partner)!: :sparkles: hero image per room type, offers grouped by room (#715)
- feat(partner)!: :boom: partner-app is gRPC-only, with a build-time OpenAPI spec (#714)
- feat(partner): :sparkles: add the Inventory gRPC surface, prices-only (#712)
- refactor(partner): :recycle: adopt a Partner protobuf naming convention and enforce it (#711)
- feat(partner): :sparkles: add the Search gRPC surface, move the billable account to metadata, and retire deprecated ScoreSort aliases (#710)
- feat(partner): partner media retrieval API with per-hotel metering (#693)
- feat(open-api): document the gRPC surface by generating OpenAPI from protobuf descriptors (#669)
- feat(partner): migrate GetLookup and establish the Stripe-style expand convention (#668)
- feat(partner): first business gRPC endpoint — destination autocomplete + lookup contract trim (#666)
- feat(grpc): :sparkles: add Spring gRPC foundation as a shared module family (#660)

