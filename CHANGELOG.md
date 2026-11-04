# Changelog

All notable changes to the Partner API protobuf contracts are documented here. Format loosely
follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow
[Semantic Versioning](https://semver.org/) as described in [README.md](README.md#versioning--release-process).

Entries below `## [Unreleased]` are populated automatically by `publishPartnerProtoContracts.bash`
in `monorepo-java` when it cuts a release; do not hand-edit past entries.

## [Unreleased]

## [2.0.0] — 2026-11-04

Released from `monorepo-java` 13c8ceb7ec1e.

### Breaking

- publish source-currency amounts with display-only FX quotes (#1332)
- share static content and paged reviews between the public and Partner APIs (#1317)
- 7c-3 channelInventoryIdentifier leaves every public contract
- remove channel_inventory_id from the Partner API; the server resolves the add-on's channel

### Added

- partner commission excludes the actual Payment Collection Fee
- partner room-type rate calendar RPC
- add SearchByFilters, SearchLatest and SearchBySavedList to the Partner Search API (#1356)
- serve venue, contact, metadata and announcement fields on inventory/content (#1352)
- add bathroomCount, securityFeatures and other guest room fields to ROOM_TYPE content (#1337)
- bring back the ADD_ON content scope and link priced offers by addOnId (#1336)
- sort search by a real price band instead of star rating (#1334)
- optional recognitions and green index scores on public and Partner content (#1330)
- apply an add-on to one, several or all items of its inventory type (#1267)
- apply mandatory add-ons on every sales channel
- phase 6 PR 2b — hotel search on PropertyAggregate

### Changed

- describe mandatory add-on quantity by hotel child-age policy

### Fixed

- degrade a property whose currency has no platform quote instead of failing the page (#1358)
- list every perk on a master rate and deprecate the unpopulated Partner perks dictionary (#1325)
- apply room type and rate plan restrictions to venue add-ons and at checkout (#1266)

## [1.1.0] — 2026-09-14

Released from `monorepo-java` 5407b4b75e9d.

### Added

- expose feeModel/acquiringFee/refundFee on BookingLightweight, LedgerBalanceResponse.withdrawable, property email Stripe fee rows


## [1.0.1] — 2026-09-08

Released from `monorepo-java` 33ba42254b62.

### Fixed

- correct the day-of-week on the taxonomy and booking `Sunset` headers

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

