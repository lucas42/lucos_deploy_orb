# ADR-0003: Skip npm release when no behaviour-affecting files have changed

**Date:** 2026-05-08
**Status:** Accepted
**Author:** lucos-architect[bot]
**Related issues:** #166 (parent decision), #116 (prior `calc-version` boundary), #97 (npm/git tag divergence — historical context), #169 (Option A implementation), #167 (Option C rollout)

## Context

The orb's `release-npm` and `release-npm-and-docker` jobs publish to npm — and tag the commit, push to Docker Hub (in the combined job), and emit a Loganne `publishedComponent` event — on every merge to `main`. Three lucos library repos use this path:

- `lucos_navbar`
- `lucos_time_component`
- `lucos_search_component`

These repos cut a new patch release on most days, almost always driven by Dependabot bumps that touch only **transitive devDependencies** (e.g. `webpack-sources` 3.4.0 → 3.4.1 in `lucos_time_component`'s lockfile). In the typical case the published artifact is byte-identical to the previous release.

**Concrete evidence from #166:** `lucos_time_component` shipped tags `v2.1.88` → `v2.1.97` (10 patch releases) in roughly two weeks. The package's `main` is `index.js` published raw with no build step; its only deps are `jest` and `webpack-cli` (both devDeps). Every one of those releases pushed the same source bytes to npm.

### Operational cost

Per `lucos-site-reliability` on consultation in #166:

> Engineer-time tax from investigating cascade flaps. When 3-4 services flap simultaneously, the no-flap-tolerance rule means I have to verify each one is benign rather than dismiss the pattern. […] One transitive devDep bump → up to 4 redeploys → 4× post-restart bursts.

> A real shared-dependency failure (TfL down, contacts service degraded, mirror saturating) presents identically to a cascade. I've trained myself to dismiss "estate-wide simultaneous flap" — that's exactly when a real correlated outage would slip past.

The cost is not just notification noise — the cascade **actively masks signal we want to act on**. SRE rated the operational priority as **medium-to-high**.

Secondary effects: anything bouncing `lucos_media_manager` cuts active long-poll connections (audible to users); Loganne's bounded webhook retry budget can be eaten by deploy storms.

### Constraint from #116

#116 (closed 2026-04-17) deliberately removed the `step halt` skip path from `calc-version` because halts didn't cross job boundaries and produced phantom "successful deploys" on the CircleCI / Loganne dashboards. Any new skip mechanism must avoid recreating that problem — specifically:

- No mid-pipeline halt that relies on later jobs to honour it.
- No state where the npm registry, the git tag, the Docker registry, and the Loganne event get out of sync. Either the entire release happens or none of it does.
- Manual rebuild-deploy must still work without empty commits.

The npm publish case is different from the deploy case in one key way: when the published artifact would be byte-identical, **not publishing is the correct outcome** — that's not a phantom no-op, that's "no change to release". So the skip is legitimate provided publish, tag, and Loganne event are skipped *together*, atomically, on the basis that there is genuinely nothing to release. See "Boundary with #116" below.

## Decision

Adopt **Option A** (file-allow-list, baked into the orb) for the publish-skip mechanism, paired with **Option C** (move the npm Dependabot schedule from daily to weekly) on the three affected library repos.

### Option A — file-allow-list in the orb

A new orb command, `check-npm-release-needed`, runs after `calc-version` and before `publish-npm` in both `release-npm` and `release-npm-and-docker`. It computes the diff between the latest `v*` tag and HEAD. If every changed file matches the allow-list, the job halts cleanly via `circleci step halt`.

The allow-list is **hardcoded in the orb** (not per-repo configurable) and covers files that are not part of the published npm artifact:

- `package-lock.json` — transitive dependency lock file
- `README.md`, `CHANGELOG.md` — documentation
- Dotfiles and dot-directories (`.circleci/`, `.github/`, `.gitignore`, `.editorconfig`, etc.)
- `test/` and `tests/` — test-only code

### Option C — weekly Dependabot schedule on library repos

`schedule.interval` for the npm ecosystem is moved from `daily` to `weekly` on `lucos_navbar`, `lucos_time_component`, and `lucos_search_component`. This reduces the frequency of routine devDep bumps without affecting any other repo.

Tracked separately as #167 (`owner:lucos-system-administrator`).

### Why no per-repo override

`lucos-system-administrator` explicitly pushed back on a per-repo `.lucos-release.yml` config: drift across the estate would create more ops cost than the flexibility benefit saves. The allow-list lives in one place, evolves with the orb, and is consistent across every repo using `release-npm` / `release-npm-and-docker`. If a concrete case ever demands a per-repo override, the question reopens.

## Boundary with #116

#116 removed `step halt` from `calc-version` because halts didn't propagate across CircleCI job boundaries. The pre-#116 shape was:

1. `calc-version` halts when HEAD is already tagged.
2. The next job in the workflow (`deploy-avalon`) runs anyway — `step halt` doesn't cross job boundaries.
3. The deploy job re-derives `VERSION`, pulls the existing image, runs `docker compose up` (no-op), fires a `deploySystem` Loganne event.
4. CircleCI dashboard and Loganne activity log show a phantom "successful deploy" with nothing actually built or deployed.

The skip introduced by this ADR has a fundamentally different shape:

| | #116 (the case we removed) | This ADR (the case we add) |
|---|---|---|
| Halt scope | One step in a multi-job workflow | One step in a **single-job** workflow (`release-npm` / `release-npm-and-docker`) |
| Effect of halt | Subsequent jobs in the workflow ran anyway | All subsequent steps in the same job are skipped (publish, tag, Loganne) |
| Result if halt fires | Phantom "deployed" state; registry/tag/event out of sync with reality | Coherent "no release" state; nothing changes anywhere |
| Correct outcome | Always cut a tag; no skip | Skip publish + tag + Loganne when nothing has changed |

`circleci step halt` *does* honour the in-job stop — it's only the cross-job propagation that fails. Within a single job, the steps after `check-npm-release-needed` (`publish-npm`, `push-release-tag`, `loganne-publish`, and in the combined job `publish-docker`) are all skipped together. There is no orphan state.

This is consistent with #116's intent: don't create signals that misrepresent reality. A no-op release skipped silently is reality. A phantom successful deploy is not.

## Loganne suppression on skip

Lucas42's framing in #166:

> Only log to loganne when something notable happens. Releasing a new version of a package is notable. Not releasing a new version of a package isn't.

A "skip" event would re-noise the activity log with information that has no consumer — exactly the cost we are trying to remove. Loganne stays silent on skips.

The CircleCI job log still shows the skip clearly (the `check-npm-release-needed` command logs the changed files and the allow-list match before halting), so debuggability is preserved at the CI layer where it belongs.

## `release-npm-and-docker` shared fate

`lucos_navbar` uses `release-npm-and-docker`, which publishes both an npm package and a build-stage `scratch` Docker image whose content is just the bundled `lucos_navbar.js`. If the npm publish is skipped, the Docker publish must also be skipped — otherwise the npm registry and Docker Hub get out of sync (e.g. a Docker tag exists for a version that was never published to npm).

The skip is placed **after `calc-version` and before `publish-npm`** in both jobs. Since `circleci step halt` stops the entire job, every downstream step — including `publish-docker`, `push-release-tag`, and the trailing `loganne-publish` — is skipped atomically.

## Bundled-output edge case (accepted trade-off)

A transitive devDep update that subtly changes a *bundled* artifact's bytes — e.g. a `terser-webpack-plugin` minor that alters minification, affecting `lucos_navbar`'s or `lucos_search_component`'s rollup output — would be skipped under Option A even though the published artifact's bytes have technically changed.

This trade-off was raised by `lucos-system-administrator` in #166 and accepted by lucas42 on the architect's view that:

1. Such differences are formatting-level (whitespace, identifier renames, statement ordering inside minified output), not behavioural.
2. The next behaviour-affecting change to the source will pick up the new minifier output naturally, so no version is permanently stranded with stale bundled bytes.
3. Option B (artifact-byte comparison) is the semantically perfect answer if this trade-off ever bites in practice; the door is left open.

This is documented inline in `src/commands/check-npm-release-needed.yml` so it remains findable when somebody is investigating a "why didn't this publish?" question, not buried only in this ADR.

## Dependabot security-update independence

Slowing the npm Dependabot `schedule.interval` from `daily` to `weekly` on the three library repos affects the **version-update feed only**. Dependabot has two independent feeds:

- **Version updates** — driven by `dependabot.yml`, governed by `schedule.interval`. Routine churn.
- **Security updates** — driven by GitHub Security Advisory matches against the dep graph. Event-triggered, **independent of `schedule.interval`**.

GitHub's documentation is explicit: *"There is no interaction between the settings specified in the `dependabot.yml` file and Dependabot security alerts, other than the fact that alerts will be closed when related pull requests generated by Dependabot for security updates are merged."*

So Option C does not delay security patching. A separate verification step (per #166's notes for triage on #167) confirms that Dependabot security updates are enabled at the repo-settings level on each of the three library repos — they are.

## Implementation status

- **Option A:** implemented in #169 (commit `33c1504`), merged 2026-05-08. The new command lives at `src/commands/check-npm-release-needed.yml` and is wired into both `src/jobs/release-npm.yml` and `src/jobs/release-npm-and-docker.yml`.
- **Option C:** tracked at #167.

## Consequences

### Positive

- **Cascade pattern broken at source.** Transitive devDep bumps on the three library repos no longer publish a new version, so direct consumers (`lucos_loganne`, `lucos_media_seinn`, `lucos_notes`, `lucos_navbar` consuming `lucos_time_component`, etc.) don't get a Dependabot PR for a byte-identical patch bump.
- **SRE signal restored.** "Estate-wide simultaneous flap" can return to its true meaning — a correlated outage worth investigating — instead of being noise from an automated chain reaction.
- **Atomic skip semantics.** Publish, tag, Docker push (where applicable), and Loganne event are skipped as one unit. No orphan tags, no orphan registry pushes, no phantom successful deploys.
- **Loganne activity log stays signal-rich.** Skips are silent; releases are notable.
- **No per-repo config drift.** The allow-list is in one place, evolves with the orb, and applies uniformly.
- **Self-contained in the orb.** No registry round-trips, no extra moving parts, reuses the git tag fetch that `calc-version` already does.

### Negative

- **The bundled-output edge case is real.** A transitive devDep update that subtly changes bundled output is skipped. Mitigation: documented inline in the orb command and in this ADR; Option B remains open as an upgrade path.
- **The allow-list is a list to maintain.** Files outside the obvious set (e.g. a future linter config, a Renovate config) will need to be added explicitly. The allow-list is intentionally conservative — when in doubt, publish — so the failure mode is "extra publish", not "missed publish".
- **Manual rebuild-deploy semantics differ from #116.** Triggering a pipeline rerun on `main` after #116 always cuts a new tag for `deploy-*` jobs. For `release-npm` / `release-npm-and-docker`, a rerun on a HEAD whose only changes since the last release tag are allow-listed will silently skip — which is correct ("nothing to release") but worth being aware of when manually re-running. The CI log makes this visible.

### Risks

- **Allow-list false negatives.** If a behaviour-affecting file is added to the allow-list by mistake, a real change will be silently dropped. Mitigation: the list is small, conservative, and reviewed at the PR layer when it changes; the conservative default ("when in doubt, publish") ensures over-skipping requires deliberate misconfiguration rather than oversight.
- **Allow-list false positives.** If a non-affecting file is *not* in the allow-list, the orb will publish a no-op release. This is the pre-#166 baseline behaviour, so the worst case is "no improvement on that file" rather than "regression". Acceptable.

## Alternatives considered

### Option B — artifact-byte comparison

After the build, fetch the previously published tarball from the npm registry and compare against the new build. Skip publish + tag + Loganne if content-identical.

**Pros:** semantically perfect. Handles the bundled-output edge case correctly without an allow-list.

**Cons:**

- Per-sysadmin: an npm registry round-trip during CI is "another thing that can fail or rate-limit at inconvenient times".
- More implementation complexity (tarball fetch, decompression, deterministic comparison handling timestamps and metadata).
- The build still runs unconditionally — no CI-time saving.

**Why deferred:** Option A handles the dominant case (Dependabot-only PRs touching only `package-lock.json`) cleanly with much less moving infrastructure. The bundled-output edge case is theoretical until proven otherwise. If it ever bites, Option B is a straightforward upgrade path layered on top of (or in place of) the allow-list.

### Per-repo `.lucos-release.yml` override

Allow each library repo to declare its own allow-list in a checked-in YAML file consumed by the orb.

**Cons:** drift across the estate; a fragmented allow-list across N repos creates more ops cost than the flexibility benefit saves. Sysadmin-pushed-back. Not adopted.

### `open-pull-requests-limit: 0` per ecosystem

A Dependabot setting that suppresses version updates entirely while leaving security updates intact. Suggested for completeness in #166.

**Cons:** too aggressive. We do still want routine bumps, just less often. Option C (`weekly`) is the balanced choice.
