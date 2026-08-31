# Bamboo pipeline — publishing releases here

**For Wink engineers only.** This is not documentation for integrators consuming the contracts —
see [README.md](../README.md) for that. This describes the Bamboo setup in the internal
`monorepo-java` repository that publishes releases to this repo. The plans and the publish script
all live in `monorepo-java`; this file is kept here (rather than only in that repo's `CLAUDE.md`)
so anyone auditing what pushes to this public repo — and how — can find it by starting from this
repo, without needing `monorepo-java` access.

## The two plans

Publishing is split across two Bamboo plans, because the thing that must be true before we tag a
public release — *the contract is live in production* — is not known to the plan that builds it.

```
Wink Production › Platform  (WP-MJ)              ── builds, does NOT deploy
├─ Default Stage : job Test
└─ Install       : job Install + Build + Push  (WP-MJ-IN)
                   artifacts: image-tag, OpenAPI Partner, Partner Proto Release
                                                            │
   (production deployment runs deployApps.bash, consuming image-tag)
                                                            │
                            ▼ on successful production release
Publish partner proto contracts                  ── tags and publishes to this repo
├─ checks out monorepo-java master
├─ downloads the Partner Proto Release artifact
└─ ./publishPartnerProtoContracts.bash
```

`WP-MJ-IN` ends at *"the image exists in Artifact Registry."* Nothing it does has deployed
anything, so it must never publish here — a release tagged at that point describes a contract that
built successfully and may never go live. That is the rule `monorepo-java/CLAUDE.md` states for the
OpenAPI reference docs, and it applies identically to the wire contract:

> **Promote on DEPLOY success, not build success.** Publishing at build time makes the reference
> describe a contract that is not live yet, and a failed or rolled-back deploy leaves it
> permanently ahead.

## The artifact handoff

### What `WP-MJ-IN` must produce

The build job is the only place with a full source checkout *of the commit being built*, so it is
where the release inputs are captured. Add a task after the build:

```bash
# Snapshot of exactly what this build compiled — 8 files under travel/wink/grpc/v1/,
# which map 1:1 onto this repo's proto/ tree.
mkdir -p target/proto-release
cp -R grpc/grpc-proto/src/main/protobuf/. target/proto-release/protobuf/

# Provenance: the commit this build actually built. Same discipline as image-tag.txt.
git rev-parse HEAD > target/proto-release/commit-sha.txt

# Full subject history for the proto path only — cheap (one small path), and it lets the
# publish step compute a version bump without depending on its own checkout being deep
# enough, on the right branch, or at the right commit.
git log --format='%H %s' -- grpc/grpc-proto/src/main/protobuf \
  > target/proto-release/proto-commits.txt
```

and a third artifact definition alongside the two already on that job:

| Name | Location | Copy pattern(s) | Required |
|---|---|---|---|
| `image-tag` | `target` | `image-tag.txt` | yes — *already defined* |
| `OpenAPI Partner` | `open-api/open-api-grpc/target/openapi` | `*.json` | yes — *already defined* |
| **`Partner Proto Release`** | `target/proto-release` | `**/*` | **yes — add this** |

### What the publish plan must do

1. Check out `monorepo-java` `master` (already configured).
2. Declare an **artifact dependency** on `Partner Proto Release`, downloading it to
   **`target/proto-release`** relative to the job's working directory. The script's default input
   path is exactly that; if Bamboo is configured to land it elsewhere, pass `--release-dir`.
3. Run `./publishPartnerProtoContracts.bash` as the final task, working directory = repo root
   (the same rule as every other Bamboo task in this reactor).

## Why the artifact, and not the plan's own checkout

**The publish plan's checkout is `master` as of when the publish plan runs — which is not
necessarily the commit that was deployed.** If anything merges to `master` between the production
release and this plan's checkout, `HEAD` silently points at a commit production has never seen.

This is the same hazard `deployApps.bash` documents in its own header, and solves the same way:

> `--tag` is how CI hands the tag forward, and is what the Bamboo CD plan should use. […] CD is a
> separate plan with its own working copy, so a commit landing on the branch during a CI run
> leaves CD's HEAD pointing at a commit that run never built […]. **Triggering CD on CI success
> does not prevent this — the checkout still happens at deploy time.**

So the script treats its own checkout as **untrusted for provenance**:

| Question | Answered by | Never by |
|---|---|---|
| Which protos do we publish? | `target/proto-release/protobuf/` (the artifact) | the checkout's working tree |
| Which commit shipped? | `target/proto-release/commit-sha.txt` | `git rev-parse HEAD` |
| What changed since the last release? | `target/proto-release/proto-commits.txt`, sliced at the `.sync-state` watermark | `git log …HEAD` |

The checkout is still useful — it is where the script runs from, and it provides the `git log`
fallback if `proto-commits.txt` is ever missing from the artifact — but nothing that ends up in a
published tag is derived from it.

A second benefit falls out of this: change detection no longer diffs two `monorepo-java` SHAs. It
compares the artifact's proto tree against what is **actually published** in this repo, which is
self-correcting. If a publish ever half-fails and the watermark drifts, a tree comparison still
sees the real difference on the next run, whereas a SHA-range diff would skip that range forever.

## Script input contract

`publishPartnerProtoContracts.bash` reads everything it needs from one directory, default
`./target/proto-release` (override with `--release-dir <dir>` or `PROTO_RELEASE_DIR`):

| Path | Required | Content |
|---|---|---|
| `protobuf/` | yes | The proto tree as built. Copied verbatim onto this repo's `proto/`. Must contain at least one `.proto`. |
| `commit-sha.txt` | yes | The 40-hex `monorepo-java` commit that was deployed. Recorded in `.sync-state` and the release notes. |
| `proto-commits.txt` | no | `<sha> <subject>` per line, newest first, for the proto path only. Falls back to `git log` in the checkout if absent. |

## What the script does

1. Validates the release directory, then clones this repo (`master`, with tags).
2. Reads `.sync-state` for the last published commit and version, and the last `v*` tag.
3. **Compares the artifact tree with the published `proto/` tree. Identical → exits 0 immediately.**
   Most runs are no-ops, since most production releases do not touch the proto contracts; this is
   what makes it safe to run unconditionally on every production release.
4. Slices `proto-commits.txt` at the watermark (see below) and derives the bump from the
   [Conventional Commit](https://www.conventionalcommits.org/) subjects in that range —
   `!` → MAJOR, `feat` → MINOR, anything else → PATCH. **With no prior tag the first release is
   `v1.0.0`**, not a computed bump.
5. Stages the new tree, then runs `buf lint` and `buf breaking` (`WIRE_JSON`, from
   [`buf.yaml`](../buf.yaml)) against the previous release tag — the same check
   [`validate.yml`](../.github/workflows/validate.yml) runs on PRs. **If buf finds a breaking
   change and no commit in range carried `!`, the run hard-fails** rather than shipping a
   MAJOR-shaped change under a MINOR tag.
6. Rewrites `proto/`, prepends a `CHANGELOG.md` entry, updates `.sync-state`, commits, tags
   `vMAJOR.MINOR.PATCH`, pushes, and creates a GitHub Release with the commit subjects as notes.

`.sync-state` is advanced **only** by a successful push, which is what makes the whole step
idempotent: re-running after any failure re-derives the same decision from the same inputs.

### `.sync-state`

```json
{
  "monorepo_java_commit": "<the deployed commit — provenance only>",
  "proto_commit": "<newest proto-touching commit at this publish — the watermark>",
  "last_published_at": "<ISO 8601 UTC>",
  "version": "<the version just published>"
}
```

`proto_commit` is a separate field from `monorepo_java_commit`, and the distinction matters: the
deployed commit usually does **not** touch the proto path, so it never appears in the path-filtered
`proto-commits.txt` and cannot be used to slice a commit range. The watermark has to be the newest
commit that actually changed the contract. If the watermark is ever missing from the artifact's
log, the histories have diverged (a force-push, a rebase, an artifact built from the wrong branch)
and the script hard-fails rather than guessing a range.

The `version` field is informational — the authoritative previous version is the newest `v*` git
tag, so a hand-edited `.sync-state` cannot make the script skip or repeat a version.

## Failure semantics

The publish runs *after* production is already live, so "fail the build" and "leave the mirror
stale" are both real costs. The split follows the allowlist principle already used by
`pushImages.bash`'s registry retries — retry only what is known-transient, fail fast on everything
else:

| Failure | Result | Why |
|---|---|---|
| Missing/malformed artifact, bad SHA, empty proto tree | **exit 1** | The handoff is broken. Publishing anything here would be guesswork. |
| `buf breaking` finds a break with no `!` commit | **exit 1** | Needs a human: either the commit convention was not followed or the change should not ship. |
| `buf lint` failure | **exit 1** | The contract is malformed; it must not reach partners. |
| Push rejected (non-fast-forward), or auth denied | **exit 1** | Someone or something else wrote to this repo, or the token is wrong. Retrying cannot fix either. |
| Network / GitHub 5xx on clone, push, or release creation | **exit 0**, logged as `PUBLISH-DEFERRED` | Nothing was written, `.sync-state` did not advance, and the next production release republishes the same diff. Failing a good production deploy over a GitHub outage is worse. |

`PUBLISH-DEFERRED` is a deliberately greppable marker — wire a Bamboo log parser or notification
to it, because a deferred publish is otherwise a green build that published nothing. If proto
changes are rare, that state can persist for weeks before the next release clears it.

## Secrets and agent prerequisites

| Variable | Value | Notes |
|---|---|---|
| `PARTNER_PROTO_REPO_TOKEN` | GitHub token scoped `contents:write` on **only** `wink-travel/partner-api-proto` | Bamboo **secret** variable, not plain. Used for both the `git push` and `gh release create` (exported as `GH_TOKEN` inside the script — one secret, not two). A fine-grained PAT or a GitHub App installation token both work; do **not** reuse a broader `repo`-scoped token that can push to other `wink-travel` repositories. |

| Tool | Why the publish plan needs it | Already required by |
|---|---|---|
| `git` | Clone, commit, tag, push | everything |
| `buf` | Lint + breaking-change gate before publishing | `grpc/grpc-proto`'s `buf breaking` gate, bound to `mvnd verify` |
| `gh` | Creates the GitHub Release after the tag is pushed | — **not otherwise required**; if the agent image lacks it, provision it the same way as `buf` (single static binary from the `gh_*` release asset, put on `PATH`) |

No `jq`: the script parses only the `.sync-state` file it writes itself, with `sed`, so the agent
needs no JSON tooling.

## Testing a change to the script without publishing anything

```bash
./publishPartnerProtoContracts.bash --dry-run --release-dir /path/to/proto-release
```

Computes and prints the version bump, the changelog entry and the file changes; runs the buf gate;
pushes nothing and creates no release. Needs no token while this repo is public.

`publishPartnerProtoContracts.test.bash` covers the decision logic with stubbed `git`/`buf`/`gh`,
in the same style as `pushImages.test.bash`:

```bash
./publishPartnerProtoContracts.test.bash
```

## Status

**Not yet wired up.** As of 2026-08-31:

- [ ] `Partner Proto Release` artifact definition + capture task on `WP-MJ-IN`
- [ ] Artifact dependency on the publish plan, downloading to `target/proto-release`
- [ ] `PARTNER_PROTO_REPO_TOKEN` created as a Bamboo secret variable
- [ ] `gh` confirmed on the publish plan's agent
- [ ] A `Public proto contracts repo (partner-api-proto)` section added to
      `monorepo-java/CLAUDE.md` — that file, not this one, is what engineers actively editing the
      pipeline will keep current, and it currently does not mention this repo at all

Until the first release lands, this repo has no `v*` tag, `.sync-state` is `null`, and `proto/`
holds only a placeholder README.
