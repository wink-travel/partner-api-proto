# Wink Partner API — Protocol Contracts

Protobuf wire contracts for the [Wink](https://wink.travel) Partner Integrator API
(`wink.partner.v1.*`). This repository is a **read-only mirror** of the `.proto` sources that
back the gRPC surface served at `partner.wink.travel:443`.

> **Status: pre-release.** No version has been published yet. The Partner gRPC API has not
> completed its production cutover — see [Versioning & release process](#versioning--release-process)
> below for what triggers the first tagged release.

## What's here

```
proto/travel/wink/grpc/v1/*.proto
```

One package, `wink.grpc.v1`, holding a cross-surface `Diagnostics` contract plus the Partner
domain contracts, each declaring its own wire package (`wink.partner.v1`) so it versions
independently of the shared package name.

This repo carries **only** the `.proto` source files — no generated language stubs. Generate your
own client with [`buf`](https://buf.build/docs/introduction) or `protoc` in whatever language you
need; the contracts depend on the standard [`googleapis`](https://buf.build/googleapis/googleapis)
common types (`google.api.http`, `google.api.annotations`, `google.type.*`), declared as a `buf.yaml`
dependency rather than vendored.

```bash
# using buf (recommended)
buf generate --template buf.gen.yaml   # bring your own buf.gen.yaml for your target language

# using protoc directly
protoc -I proto --java_out=... proto/travel/wink/grpc/v1/*.proto
```

## Human-readable API reference

This repo is the wire contract, not the documentation. For the browsable REST/JSON-shaped API
reference (generated from these same protobuf descriptors at build time), see the
[Partner API docs](https://wink.travel/partner-api/partner), published from `monorepo-java`'s
`open-api/open-api-grpc` module. That page goes live with the next release — see
[Versioning & release process](#versioning--release-process) below.

## Versioning & release process

Releases are tagged `vMAJOR.MINOR.PATCH` and published only after a change has actually shipped to
**production** — never on every merge to the internal `develop` branch. That mirrors this
platform's existing rule for its OpenAPI reference docs: a published contract should describe what
is live, not what is merged.

Each release is a single squashed commit representing the proto tree as of that production deploy,
tagged and attached to a [GitHub Release](../../releases) with a changelog. This repo does not
carry the internal monorepo's full commit history — see [CONTRIBUTING.md](CONTRIBUTING.md) for why.

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
