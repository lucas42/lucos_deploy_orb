# ADR-0002: Two-config-file approach for orb integration testing

**Date:** 2026-04-18
**Status:** Accepted
**Author:** lucos-developer[bot]
**Related issues:** #124

## Context

On 2026-04-17 a bug in `publish-docker.yml` shipped to production undetected: the command still used `docker tag` + `docker push` to create the `:latest` tag, but the surrounding buildx `docker-container` driver never loads the built image into the host daemon, so `docker tag` failed immediately. The orb's own CI didn't catch this because there was no integration test exercising the `:latest` publish path.

### Why single-pipeline testing doesn't work

CircleCI resolves all orb references **at the start of a workflow**, before any jobs run. This means a job that calls `deploy/publish-docker` from `lucos/deploy@dev:alpha` will always use the version that existed when the pipeline started — not the version that `orb-tools/publish-dev` publishes in the same pipeline.

In practice: any change to `publish-docker.yml` won't be caught by an integration test in the same pipeline. There is an unavoidable **one-pipeline lag** regardless of `requires:` ordering.

A previous attempt (PR #127) went through several iterations trying to work around this — inline Docker commands, `requires: [orb-tools/publish-dev]` ordering, bootstrap "trigger commit" approaches — and all hit the same constraint. The bootstrap approach eventually produced green CI but relied on a deliberately empty trigger commit and SHA deduplication behaviour in CircleCI, making it brittle and hard to understand.

## Decision

Use CircleCI's **dynamic configuration** feature (`setup: true`) with the `circleci/continuation` orb.

- `config.yml` — configured as a setup workflow. Lints, packs the orb, publishes `dev:alpha` on all branches, then triggers `test-deploy.yml` as a pipeline continuation.
- `test-deploy.yml` — separate continuation config. Imports `lucos/deploy@dev:alpha` (which now definitely exists when this pipeline starts), runs a `test-publish-docker` job against a real Docker Hub push, then (on main only) runs `orb-tools/increment` gated on test success.

Because `test-deploy.yml` is a *new pipeline* (a continuation), it resolves `lucos/deploy@dev:alpha` fresh — after `config.yml` has already published it. The lag is eliminated.

### How continuation works

CircleCI injects `CIRCLE_CONTINUATION_KEY` automatically into any setup pipeline. The `continuation/continue` step uses this key to hand off to the next config file. No external personal API token (`CCI_TOKEN`) is needed. This is the pattern recommended by CircleCI's own orb author documentation.

### The test job

`test-publish-docker` in `test-deploy.yml`:

1. Creates a minimal `Dockerfile` (FROM alpine) and a `docker-compose.yml` with `image: lucas42/deploy-orb-test:${VERSION}` where `VERSION=v${CIRCLE_PIPELINE_ID}` (unique per run).
2. Calls `deploy/publish-docker` — the real orb command, not an inline copy of it.
3. Verifies the versioned tag exists via `docker buildx imagetools inspect`.
4. On main only: verifies `:latest` was also pushed.

Using `CIRCLE_PIPELINE_ID` as the tag suffix ensures concurrent pipeline runs don't clobber each other's verification. Test images push to `lucas42/deploy-orb-test` (a dedicated Docker Hub repository for this purpose).

### No `force-latest` parameter

A `force-latest` parameter was proposed to allow `:latest` to be pushed on non-main branches (so the test could verify it on feature branches). This was explicitly rejected by the repository owner: it introduces a test-only code path, meaning we would no longer be testing the real publish flow.

Instead, `:latest` verification only runs on the main branch (where the real code pushes it). On non-main branches, the test verifies only the versioned tag — which is still valuable and catches the class of bug seen in 2026-04-17.

## Consequences

**Benefits:**
- Integration tests run against the actual orb code in every pipeline, including on main before `increment` publishes a new semver version.
- A broken `:latest` push path on main will block the orb from being published to the estate.
- No test-only code paths in the orb itself.
- No external `CCI_TOKEN` required.

**Trade-offs:**
- CI now has two config files, which adds some complexity.
- `setup: true` pipelines have a slightly different structure from standard CircleCI pipelines — maintainers need to understand the continuation model.
- `orb-tools/pack` runs twice on main (once in `config.yml` for `publish-dev`, once in `test-deploy.yml` for `increment`). This is a minor inefficiency accepted for architectural simplicity.
- The `:latest` push path is only integration-tested on main, not on feature branches. A regression on a feature branch would surface on main CI (gated before `increment`), not before merge.
