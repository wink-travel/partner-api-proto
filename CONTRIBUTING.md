# Contributing

**This repository is a generated mirror.** The source of truth for these contracts is
`grpc/grpc-proto/src/main/protobuf` in Wink's internal `monorepo-java` repository. Every commit on
`master` here is produced by an automated release job (`publishPartnerProtoContracts.bash`) that
runs as part of that repo's production Bamboo pipeline — see `monorepo-java`'s own history (feature
branches merge to `develop`, releases cut to `master` via GitFlow) for the authoritative PR
discussion and design rationale behind every field.

## Why not mirror full history?

The internal repo's commit history references internal ticket numbers, other modules, and
sometimes internal implementation detail in commit bodies. Squashing each release to a single
commit here keeps this repo's history scoped to exactly what a partner integrator needs: what the
contract looked like at each released version, and why it changed (see each tag's
[Release notes](../../releases)).

## Reporting a problem with the contract

Open an issue here describing the problem. Do not open a PR modifying the `.proto` files directly
— it will be overwritten by the next automated sync and won't reach the people who own the
service implementation. An engineer will pick it up and make the change in `monorepo-java`, where
it goes through the normal review + `buf breaking` gate before being mirrored out here.

## Local validation

PRs to this repo (documentation, README, CI config — not proto files) are linted with `buf lint`
and `buf breaking` against the previous release via `.github/workflows/validate.yml`.
