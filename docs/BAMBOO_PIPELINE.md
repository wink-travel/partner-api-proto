# Bamboo build plan — publishing releases here

**For Wink engineers only.** This is not documentation for integrators consuming the contracts —
see [README.md](../README.md) for that. This describes the Bamboo setup in the internal
`monorepo-java` repository that publishes releases to this repo. The plan, its stages, and the
publish script all live in `monorepo-java`; this file is kept here (rather than only in that
repo's `CLAUDE.md`) so anyone auditing what pushes to this public repo — and how — can find it by
starting from this repo, without needing `monorepo-java` access.

## Where this fits in the existing plan

`monorepo-java` already has a **Production — Release → Master** Bamboo plan (three stages: Test,
Install, Build-and-push-images). This adds a fourth stage to that same plan — it does **not** need
its own plan, since it only ever needs to run as a tail step of a production release and always
needs that release's commit checked out.

```
Production — Release → Master
├─ Stage 1: Test              (mvnd test -Pprod, full reactor, no cache)
├─ Stage 2: Install           (mvnd install -Pprod -DskipTests)
├─ Stage 3: Build & push images (spring-boot:build-image, all four apps)
└─ Stage 4: Publish partner proto contracts   <-- this repo receives its releases from here
```

## Stage 4 definition

**Job**: single shell task, working directory = repo root (same rule as every other Bamboo Maven
task in this reactor — see `monorepo-java/CLAUDE.md`'s CI Pipeline section for why).

**Task**:
```bash
./publishPartnerProtoContracts.bash
```

**Required plan/stage variables** (Bamboo secret variables, not plain — these are credentials):

| Variable | Value | Notes |
|---|---|---|
| `PARTNER_PROTO_REPO_TOKEN` | a GitHub token scoped to `contents:write` on **only** `wink-travel/partner-api-proto` | Used for both the `git push` and the `gh release create` call (exported as `GH_TOKEN` inside the script — one secret, not two). A fine-grained PAT or a GitHub App installation token both work; do **not** reuse a broader `repo`-scoped token that can push to other `wink-travel` repositories — this stage should not be able to touch anything but this one repo. |

**Agent prerequisites** — both already provisioned for other stages in this same plan, nothing new
to install for Stage 4 specifically:

| Tool | Why Stage 4 needs it | Already required by |
|---|---|---|
| `buf` | Cross-checks the proto diff against the WIRE_JSON breaking-change policy before publishing | `grpc/grpc-proto`'s `buf breaking` gate, bound to `mvnd verify` (Stage 1) |
| `gh` (GitHub CLI) | Creates the GitHub Release after the tag is pushed | — **not otherwise required today**; if the Bamboo agent image doesn't already carry it, add a one-time provisioning step (single static binary, same pattern as the `buf` install: download the `gh_*_linux_amd64.tar.gz` release asset, put `gh` on `PATH`) |

**Trigger — this is the part that's easy to get wrong.** Wire Stage 4 to run only after Stage 3's
work has actually reached production, not merely after the images were built and pushed to the
registry. Concretely:

- If Stage 3 (`spring-boot:build-image ... -Dspring-boot.build-image.publish=true`) *is* the deploy
  — i.e. pushing the image to the registry Cloud Run watches is what makes it live — Stage 4 can
  run immediately after Stage 3 succeeds.
- If there is a **separate** deploy stage/job/plan downstream of Stage 3 (a Cloud Run deploy step,
  a manual promotion gate, anything that isn't "image exists in the registry"), Stage 4 must depend
  on *that* succeeding instead. Publishing on image-push alone would tell partners about a contract
  that built successfully but never actually went live — see `monorepo-java/CLAUDE.md`'s note on
  why the OpenAPI-docs pipeline was redesigned around this exact same failure mode ("promote on
  deploy success, not build success").

Check how `deployApps.bash` / the production Cloud Run deploy is actually wired into this Bamboo
plan before setting Stage 4's dependency — this document does not assume that answer, because it
depends on Bamboo plan configuration this repo can't see.

## What the stage actually does

`publishPartnerProtoContracts.bash` (full comments at the top of the script itself):

1. Clones this repo, reads `.sync-state` for the `monorepo-java` commit SHA of the last publish.
2. Diffs `grpc/grpc-proto/src/main/protobuf` between that SHA and the current `HEAD`. **No diff →
   exits 0 immediately** — most Stage 4 runs will be no-ops, since most production releases don't
   touch the proto contracts. This makes it safe to run unconditionally on every production
   release rather than needing its own change-detection gate upstream.
3. Scans the commit subjects in that range for the Conventional Commit `!` breaking marker this
   API already uses, computing MAJOR / MINOR / PATCH.
4. Cross-checks with `buf breaking` (WIRE_JSON) against the last-published snapshot; hard-fails if
   buf finds a breaking change with no `!` commit in range, rather than silently under-versioning it.
5. Copies the proto tree into `proto/`, updates `CHANGELOG.md` and `.sync-state`, commits, tags
   `vMAJOR.MINOR.PATCH`, pushes `master` and the tag, and creates a GitHub Release with the commit
   list as notes.

## Testing a change to the script without publishing anything

```bash
DRY_RUN=1 PARTNER_PROTO_REPO_TOKEN=<a token with at least read access> ./publishPartnerProtoContracts.bash
```
Computes and prints the version bump and changelog entry; pushes and creates no release.

## Status

**Not yet wired up.** As of this writing, Stage 4 does not exist in the Bamboo plan yet, and
`PARTNER_PROTO_REPO_TOKEN` has not been created. See `monorepo-java/CLAUDE.md`'s
"Public proto contracts repo (partner-api-proto)" section for the up-to-date status — that file,
not this one, is the one engineers actively editing that pipeline will keep current.
