# Wink Partner API — Protocol Contracts

Protobuf wire contracts for the [Wink](https://wink.travel) Partner Integrator API
(`wink.partner.v1.*`). This repository is a **read-only mirror** of the `.proto` sources that
back the gRPC surface served at `partner.wink.travel:443`.

> **Status: released.** The first tagged contract, `v1.0.0`, shipped on 2026-09-02 alongside the
> Partner gRPC API's production cutover. See [Releases](../../releases) for the current version and
> [CHANGELOG.md](CHANGELOG.md) for what changed in each one.

## What's here

```
proto/wink/grpc/v1/       — wink.grpc.v1     (cross-surface)
proto/wink/partner/v1/    — wink.partner.v1  (Partner domain)
```

Two proto packages. `wink.grpc.v1` holds the cross-surface `Diagnostics` contract shared with
Wink's other gRPC surfaces; `wink.partner.v1` holds the Partner domain itself, so it versions
independently. Every file lives at a directory path matching its own proto package — buf's
`PACKAGE_DIRECTORY_MATCH` and `DIRECTORY_SAME_PACKAGE` rules are enforced, not excepted (see
[`buf.yaml`](buf.yaml)).

| Service | Package | File |
|---|---|---|
| `Diagnostics` | `wink.grpc.v1` | `proto/wink/grpc/v1/diagnostics.proto` |
| `Accounts` | `wink.partner.v1` | `proto/wink/partner/v1/partner_account.proto` |
| `Booking` | `wink.partner.v1` | `proto/wink/partner/v1/partner_booking.proto` |
| `Content` | `wink.partner.v1` | `proto/wink/partner/v1/partner_content.proto` |
| `Inventory` | `wink.partner.v1` | `proto/wink/partner/v1/partner_inventory.proto` |
| `Lookup` | `wink.partner.v1` | `proto/wink/partner/v1/partner_destination_lookup.proto` |
| `Search` | `wink.partner.v1` | `proto/wink/partner/v1/partner_search.proto` |

Shared messages and enums used across the Partner services live in
`proto/wink/partner/v1/partner_common.proto`, which declares no service of its own.

Note that service names carry no `Service` suffix (`Inventory`, not `InventoryService`). That is
deliberate and permanent: the name is part of the gRPC method path clients route on
(`/wink.partner.v1.Inventory/GetX`), so renaming it would be a breaking change.

## Generating a client

This repo carries **only** the `.proto` source files — no generated language stubs. Generate your
own client with [`buf`](https://buf.build/docs/introduction) or `protoc` in whatever language you
need. The contracts depend on the standard [`googleapis`](https://buf.build/googleapis/googleapis)
common types (`google.api.http`, `google.api.annotations`, `google.type.*`), declared as a
[`buf.yaml`](buf.yaml) dependency rather than vendored.

```bash
# using buf (recommended) — resolves the googleapis dependency for you
buf generate --template buf.gen.yaml   # bring your own buf.gen.yaml for your target language

# using protoc directly — you must supply googleapis yourself on the include path
protoc -I proto -I path/to/googleapis --java_out=out \
  $(find proto -name '*.proto')
```

Pin to a tag rather than tracking `master`, so a later release can't change your generated stubs
out from under you:

```bash
git clone --branch v1.0.0 --depth 1 https://github.com/wink-travel/partner-api-proto.git
```

## Human-readable API reference

This repo is the wire contract, not the documentation. For the browsable REST/JSON-shaped API
reference (generated from these same protobuf descriptors at build time), see the
[Partner API docs](https://wink.travel/partner-api/partner), published from `monorepo-java`'s
`open-api/open-api-grpc` module.

## Versioning & release process

Releases are tagged `vMAJOR.MINOR.PATCH` and published only after a change has actually shipped to
**production** — never on every merge to the internal `develop` branch. That mirrors this
platform's existing rule for its OpenAPI reference docs: a published contract should describe what
is live, not what is merged.

Each release is a single squashed commit representing the proto tree as of that production deploy,
tagged and attached to a [GitHub Release](../../releases) with a changelog. This repo does not
carry the internal monorepo's full commit history — see [CONTRIBUTING.md](CONTRIBUTING.md) for why.
The [`.sync-state`](.sync-state) file records which `monorepo-java` commit each published tree came
from.

The version bump is derived automatically from the
[Conventional Commit](https://www.conventionalcommits.org/) subjects of every internal commit that
touched the proto sources since the last release, the same convention already used internally for
this API (`feat(partner)!: ...`, `fix(partner): ...`):

| Commit carries | Bump |
|---|---|
| A `!` after the type/scope (e.g. `feat(partner)!:`, `fix(partner)!:`) | **MAJOR** |
| `feat` (no `!`) | **MINOR** |
| Anything else (`fix`, `refactor`, `docs`, ...) | **PATCH** |

As a safety net, every release run also cross-checks the proto diff with
[`buf breaking`](https://buf.build/docs/breaking/overview) (category `WIRE_JSON` — a JSON field
*rename* counts as breaking here, since the wire contract is also served as JSON via the OpenAPI
doc, even though raw protobuf binary compatibility wouldn't care). If `buf` finds a breaking change
that no commit marked with `!`, the release fails rather than silently shipping a MAJOR-shaped
change under a MINOR tag — see `publishPartnerProtoContracts.bash` in `monorepo-java`.

## This repo is generated — don't edit here

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[Apache License 2.0](LICENSE).
